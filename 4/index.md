# 逆向AES加密脚本

<!--more-->

## Nodejs进行AES加密

{{< admonition>}}

本地需要Nodejs环境

{{< /admonition>}}

```js
var CryptoJS = require("crypto-js");
// 只需找到key和iv
var key;
var iv;
function encrypt(data) {
    encryptedKey = CryptoJS.AES.encrypt(data, key, 
    {
        iv: iv,
        mode: CryptoJS.mode.CBC,
        padding: CryptoJS.pad.Pkcs7
    }).toString()
    return encryptedKey
}
```

## python脚本调用

```python
import execjs
result = execjs.compile(open('test.js','r').read()).call('encrypt','test')
print(result)
```

