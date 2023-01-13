# 三行代码解决验证码


<!--more-->

## 安装依赖

```shell
pip install ddddocr
```

## 实现代码

```python
import ddddocr

ocr = ddddocr.DdddOcr(show_ad=False)
# 通过img_base64识别验证码，img_base64不包含数据头
res = ocr.classification(img_base64)
# 通过img_bytes识别验证码，将图片下载到本地识别验证码
with open('1.png', 'rb') as f:
    img_bytes = f.read()
res = ocr.classification(img_bytes)

print(res)
```

{{< admonition info>}}

参考：https://cloud.tencent.com/developer/article/1853149

{{< /admonition>}}
