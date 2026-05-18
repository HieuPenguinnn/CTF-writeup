# web/vibecoded

Message gợi ý:

> Source: true me bro, anyways um this is an instance i hope so people other than your team can't see messages...

Điều này gợi ý rằng app có cơ chế phân tách dữ liệu theo từng instance/team.

Khi truy cập instance của team, ví dụ:

```text
https://vibecoded-ed14c7d1a97f5ab9.tjc.tf/
```

browser sẽ gửi request với header:

```http
Host: vibecoded-ed14c7d1a97f5ab9.tjc.tf
```

App dùng `Host` header để xác định instance. Nếu ta sửa `Host` thành domain canonical:

```http
Host: vibecoded.tjc.tf
```

thì app sẽ đọc nhầm dữ liệu từ instance chung thay vì instance riêng của team.

Đây là lỗi **Host Header Trust / Instance Isolation Bypass**.

## Khai thác

Sửa Host header:

```http
Host: vibecoded-ed14c7d1a97f5ab9.tjc.tf
```

thành:

```http
Host: vibecoded.tjc.tf
```

Tìm flag trong response

![alt text](image.png)

## Flag

```text
tjctf{th1s_1s_Y_w3_d0nt_vibeeee_codeeee_sv3lte_ov3r_r34ct_any_d4y_r34ct_s3rv3r_c0mp0n3nts_CVE-2025-55182}
```
