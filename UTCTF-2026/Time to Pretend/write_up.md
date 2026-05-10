# Time to Pretend

## Overview

Challenge này cho một trang đăng nhập, form không có mật khẩu, chỉ có `username`, `otp`. Mục tiêu là tìm được tài khoản còn hoạt động, tạo OTP hợp lệ, đăng nhập vào portal và lấy flag.

## Phân tích website

Khi truy cập trang web, ta thấy một form đăng nhập thông thường. Tuy nhiên khi xem source HTML của trang, có một comment khá đáng chú ý:

```html
<!-- NOTICE to DEVS: login currently disabled, see /urgent.txt for info -->
```

Dòng này cho biết có một file nội bộ tại đường dẫn `/urgent.txt` chứa thông tin quan trọng.

## Nội dung file `/urgent.txt`

Khi truy cập `/urgent.txt`, ta nhận được nội dung như sau:

```text
URGENT - READ IMMEDIATELY
=========================
TO: dev team
FROM: timothy
DATE: 2013-11-12 03:47:22

guys,

i think someone figured out the AffinKey system. i dont have time to explain everything
right now but there is a SERIOUS flaw in how we generate the OTPs. i only just realized
it tonight and i am freaking out.

i have locked every account in the system except mine while we figure this out. DO NOT
unlock anyone until we have patched this. i dont care if users complain. i dont care if
chad emails again. nobody gets in.

my account stays active because i need access to keep monitoring the situation. if you
need to reach me use signal.

do NOT roll back the auth system yet - i need to look at the logs first to see if anyone
has already gotten in. if they have we have a much bigger problem.

i will write up a full post-mortem once i stop shaking.

do not talk to anyone about this. not on slack. not on email. definitely not on the forum.

- timothy

p.s. i know kevin is going to say "i told you so" about building this ourselves.
he was right. i don't want to hear it.
```

Từ nội dung trên, có thể rút ra hai ý quan trọng:

1. Hệ thống OTP có lỗ hổng nghiêm trọng.
2. Tất cả tài khoản đều đã bị khóa, chỉ còn tài khoản của `timothy` là còn hoạt động.

## Phân tích file `afteahLEAK.pcap`

Tiếp theo, mở file `afteahLEAK.pcap` bằng Wireshark và kiểm tra lưu lượng HTTP. Trong đó có nhiều request gửi đến endpoint:

```text
POST /debug/getOTP
```

Dữ liệu gửi đi có dạng:

```json
{"username": "carrasco", "epoch": 1773290571}
```

Phản hồi trả về có dạng:

```json
{
  "add": 13,
  "mult": 7,
  "otp": "bnccnjbh"
}
```

Tức là với mỗi `username` và `epoch`, server sẽ trả về 3 giá trị:

* `add`
* `mult`
* `otp`

Nhận ra đây là Affine Cipher. Công thức mã hóa Affine Cipher là:

```text
E(x) = (mult * x + add) mod 26
```

Sau khi đối chiếu nhiều gói tin trong file pcap, ta có các cặp dữ liệu sau:

```python
{'username': 'carrasco', 'epoch': 1773290571} -> {'add': 13, 'mult': 7, 'otp': 'bnccnjbh'}
{'username': 'mix', 'epoch': 1773290574} -> {'add': 16, 'mult': 15, 'otp': 'ogx'}
{'username': 'hebert', 'epoch': 1773290575} -> {'add': 17, 'mult': 17, 'otp': 'ghihuc'}
{'username': 'monks', 'epoch': 1773290576} -> {'add': 18, 'mult': 19, 'otp': 'myfaw'}
{'username': 'eyre', 'epoch': 1773290577} -> {'add': 19, 'mult': 21, 'otp': 'zdmz'}
{'username': 'jurado', 'epoch': 1773290579} -> {'add': 21, 'mult': 25, 'otp': 'mbevsh'}
```

Sau khi kiểm tra quy luật, ta suy ra được:

```python
add = epoch % 26
```

Các giá trị `mult` hợp lệ lần lượt là:

```python
[1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, 25]
```

Và cách chọn là:

```python
mult = valid_mults[epoch % 12]
```

## Script khai thác

Script Python dùng để sinh OTP hợp lệ cho tài khoản `timothy`:

```python
import requests, time

valid_mults = [1, 3, 5, 7, 9, 11, 15, 17, 19, 21, 23, 25]

def gen_otp(username, epoch):
    add = epoch % 26
    mult = valid_mults[epoch % 12]
    return "".join(
        chr((mult * (ord(c) - ord('a')) + add) % 26 + ord('a'))
        for c in username
    )

epoch = int(time.time())
otp = gen_otp("timothy", epoch)

print(otp)

s = requests.Session()
s.post("http://challenge.utctf.live:9382/auth", json={"username": "timothy", "otp": otp})
print(s.get("http://challenge.utctf.live:9382/portal").text)
```

Sau khi đăng nhập thành công, trang `/portal` hiện ra dashboard của Timothy.

![alt text](image.png)

## Flag

```text
utflag{t1m3_1s_n0t_r3ll0b1l3_n0w_1s_1t}
```
