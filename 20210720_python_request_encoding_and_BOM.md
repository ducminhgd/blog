# Python `requests` encoding và UTF8-BOM

Hôm nay support đàn em làm cái vụ subtitle, nên học được thêm một số thông tin như sau:

## Python `requests`

Ví dụ:

```python
import requests
resp = requests.get('https://example.com/link-text-file.vtt')
print(resp.headers.get('Content-Type')) # text/vtt
print(resp.apparent_encoding) # utf-8
print(resp.encoding) # ISO-8859-1
```

Theo như tìm hiểu từ Internet, thì có một số kết luận như sau:

1. Nếu như trong response header `Content-Type` không có `charset` thì `encoding` mặc định hiểu là `ISO-8859-1`. Nếu `Content-Type: text/vtt; charset=utf-16` thì `encoding` sẽ là `utf-16`.
1. `apparent_encoding` thì dự vào nội dung của file để lấy ra charset.

## UTF8 with and without BOM

Thật sự mình từ lâu ghét cái BOM trong UTF-8. Lời giải thích ngắn ngọn: https://stackoverflow.com/a/2223926/701950