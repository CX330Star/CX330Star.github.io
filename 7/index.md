# TG机器人小试牛刀


<!--more-->

## 环境搭建

由于机器人后台脚本需要持续运行，推荐使用VPS，本篇环境为linux环境

epic页面存在动态加载，通过selenium控制浏览器的方式来实现

### 安装Chrome浏览器

linux安装Chrome浏览器，参考：https://support.google.com/chrome/a/answer/9025903（官方教程较为详细麻烦）

省事一点可执行以下命令

```shell
# 适用于 Debian 和 Ubuntu 平台
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
dpkg -i google-chrome-stable_current_amd64.deb
# 适用于 Fedora 和 openSUSE 平台
wget https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
rpm -ivh google-chrome-stable_current_x86_64.rpm
```

### 安装chromedriver驱动

根据Chrome浏览器版本，安装对应的chromedriver驱动，下载地址：https://chromedriver.chromium.org/downloads

```shell
# 查看google-chrome版本
google-chrome --version
wget https://chromedriver.storage.googleapis.com/对应版本/chromedriver_linux64.zip
unzip chromedriver_linux64.zip
# 将chromedriver移动到/usr/bin/目录下，LICENSE.chromedriver用不到可以删除
mv chromedriver /usr/bin/
# 查看chromedriver版本
chromedriver --version
```

### 安装需要依赖的包

```shell
pip install selenium --upgrade
pip install python-telegram-bot --upgrade
```

### 在 Telegram 中创建机器人

参考：https://cloud.google.com/dialogflow/es/docs/integrations/telegram?hl=zh-cn

1. 登录 Telegram 并转到 https://telegram.me/botfather
2. 点击网页界面中的 **Start** 按钮或输入 /start
3. 点击或输入 **/newbot** 并输入名称
4. 为聊天机器人输入一个用户名，该名称应以“bot”结尾（例如 garthsweatherbot）
5. 复制生成的访问令牌

## 具体代码

机器人代码如下:

参考文档：

python-telegram-bot官网：https://python-telegram-bot.org/

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from telegram import Update
from telegram.ext import ApplicationBuilder, CommandHandler, ContextTypes
import re

# 获取epic免费游戏信息
async def getEpicInfo(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text('正在查询...')
    chrome_options = Options()
    user_agent = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/12.0.3 Safari/605.1.15'
    chrome_options.add_argument('--headless')
    chrome_options.add_argument('--no-sandbox')
    chrome_options.add_argument('--user-agent=' + user_agent)
    browser = webdriver.Chrome(options=chrome_options)
    browser.get('https://store.epicgames.com/zh-CN/')
    lis  = re.findall(r'<a aria-label="免费游戏,(.*?)role="link',browser.page_source)
    for li in lis:
        await update.message.reply_text(li)
    browser.quit()

async def hello(update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    await update.message.reply_text(f'Hello {update.effective_user.first_name}')

def main() -> None:
    app = ApplicationBuilder().token('YOUR TOKEN HERE').build()
    app.add_handler(CommandHandler('epic', getEpicInfo))
    app.add_handler(CommandHandler('hello', hello))
    app.run_polling()

if __name__ == '__main__':
    main()
```

## 遇到问题

### headless

Chrome浏览器分为有头模式和无头模式，如果是有头模式，会弹出一个Chrome浏览器窗口，无头模式则不会弹出任何窗口，只有进程

由于VPS的linux无GUI，无法弹出Chrome浏览器窗口，selenium启动Chrome时，需添加参数`headless`，设置Chrome为无头模式，否则会出现以下报错

```
selenium.common.exceptions.WebDriverException: Message: unknown error: Chrome failed to start: exited abnormally.
  (unknown error: DevToolsActivePort file doesn't exist)
  (The process started from chrome location /usr/bin/google-chrome is no longer running, so ChromeDriver is assuming that Chrome has crashed.)
```

### no-sandbox

Chrome浏览器默认不允许root用户启动，需添加`no-sandbox`参数，关闭web沙盒，否则会出现与上面相同的报错

```
selenium.common.exceptions.WebDriverException: Message: unknown error: Chrome failed to start: exited abnormally.
  (unknown error: DevToolsActivePort file doesn't exist)
  (The process started from chrome location /usr/bin/google-chrome is no longer running, so ChromeDriver is assuming that Chrome has crashed.)
```

但此操作有可能导致浏览恶意网站时被入侵

### 无头模式易被识别为爬虫

解决方法有两种：

#### 使用无头模式

继续使用无头模式，通过添加参数进行混淆绕过爬虫检测

本篇文章的机器人便通过添加`user_agent`参数进行混淆绕过爬虫检测

#### 使用有头模式

通过`Xvfb`运行脚本，Xvfb在虚拟内存中执行所有图形操作而不显示任何屏幕输出

具体参考：https://www.kingname.info/2021/02/16/use-selenium-head-in-linux/

简单命令如下：

```shell
sudo apt-get update
sudo apt-get install xvfb
xvfb-run python3 test.py
```

{{< admonition>}}

在进行测试后发现一处问题该篇参考文章未提及：

如果当前用户是root用户，需添加`no-sandbox`参数，关闭web沙盒，才能成功运行，否则同样会报错

{{< /admonition>}}

## 参考文档

python-telegram-bot官网：https://python-telegram-bot.org/

VPS模拟浏览器有头模式：https://www.kingname.info/2021/02/16/use-selenium-head-in-linux/
