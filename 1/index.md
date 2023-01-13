# 通过Hugo和Actions快速搭建github博客


<!--more-->

## 前言

简单叙述一下流程，先在本地仓库搭建好Hugo博客环境，再在github上创建两个仓库，其中一个可为任意名称的私有库暂且称其为`Blog_Private`，另一个必须是：github用户名.github.io为命名的公开库，假设你的用户名为xiaoming，该仓库名应为：`xiaoming.github.io`，且必须为公开库

* 本地仓库
* `Blog_Private`库
* `xiaoming.github.io`库

以上三个仓库准备好之后，再进行三步配置便可完成

* 第一步，将本地仓库提交到`Blog_Private`库中
* 第二步，配置Actions文件
* 第三步，生成并设置密钥

### 思路

本地同步`Blog_Private`库，`Blog_Private`库通过Actions自动部署静态页面到公开库`xiaoming.github.io`中，通过域名xiaoming.github.io访问到公开库`xiaoming.github.io`的pages中查看博客

### 好处

该方法是把源码部署到私有库中，将自动生成的静态页面放在公开库`xiaoming.github.io`中，相较于直接将源码部署到`xiaoming.github.io`中更加安全方便

## 部署本地环境

{{< admonition>}}

**只适用于Hugo系统**

**推荐下载extended版本，该版本可自定义样式**

{{< /admonition>}}

Hugo下载地址：https://github.com/gohugoio/hugo/releases

解压到合适位置，配置hugo.exe的环境变量，方便使用命令行

这里参考LoveIt主题的文档：https://hugoloveit.com/zh-cn/theme-documentation-basics/

```shell
# 创建一个网站并进入
hugo new site my_website
cd my_website
# 通过git下载主题
# git安装参考：https://blog.csdn.net/mukes/article/details/115693833，几乎可以一路默认
git init
git submodule add https://github.com/dillonzq/LoveIt.git themes/LoveIt
# 修改config.toml配置文件，可进入https://hugoloveit.com/zh-cn/theme-documentation-basics/，作为配置参考
# 中文需要在第二行添加一条配置
defaultContentLanguage = "zh-cn"

# 生成第一篇文章，在first_post.md随意输入，draft: true 改为 draft: false
hugo new posts/first_post.md
# 本地启动测试
hugo serve
# 访问
http://127.0.0.1:1313
```

能成功访问到并能正常打开文章，本地环境便部署成功

还有很多主题可供选择：https://themes.gohugo.io/，使用时可参考Demo文档

文章开头可参考

```markdown
---
title: "test"
date: 1900-01-01
categories: ["test"]
tags: ["test"]
description: "test"
draft: false
---
<!--more-->
```

## Git同步本地和远程仓库

通过HTTPS协议连接GitHub仓库，参考：https://www.cnblogs.com/liyan-blogs/p/15153764.html

**过程中会遇到空目录不上传的问题，可在空目录中创建`.gitkeep`文件作为占位符**

**注意这里同步的是Blog_Private仓库**

```shell
# 配置git
git config --global user.name “your name”
git config --global user.email “your email”
# 在需要同步的本地仓库，执行初始化
git init
```
``` shell
# git关联远程仓库地址的三种方式
## 1.先删后加，add是需要先删处再添加，origin默认推荐仓库名
git remote remove origin
git remote add origin https://github.com/test/test.git
## 2.修改，set-url是在已有连接时修改关联远程数据库
git remote set-url <remote_name> <remote_url>
git remote set-url origin https://github.com/test/test.git
## 3.直接修改config文件
git config -e

# 获取token，在连接过程中会需要认证，选择token认证
右上角头像 --> Settings -->  Developer Settings --> Personal access tokens(classic)
```
``` shell
# 将远程仓库的文件pull到本地
git pull --rebase origin master

# 提交代码小脚本:upload.bat
@echo off
cd 本地仓库根目录
git status	# 查看代码状态
git add .	# 将代码添加到暂存区，(.代表该目录下所有文件、还可用*.md等等)
git commit -m "first commit"	# 将代码提交到本地仓库(后跟该次提交的名称)
git push origin master	# 将代码push到远程仓库
pause
```

## 配置Actions文件

{{< admonition>}}

**在Blog_Private仓库的Actions中自定义一个workflows**

**配置如下，只适用于Hugo系统**

{{< /admonition>}}

```yaml
name: GitHub Pages
on:
  push:
    branches:
      - master

jobs:
  deploy:
    runs-on: ubuntu-20.04
    steps:
      - name: Checkout repositories
        uses: actions/checkout@v2
        with:
          submodules: true
          fetch-depth: 0

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: 'latest'

      - name: Build
        run: hugo

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY_KEY }}
          external_repository: xiaoming/xiaoming.github.io	# 只需修改此处，改为自己的用户名
          publish_branch: master
          publish_dir: ./public
```

配置完成后，再次点击`Actions`，点击中间`main.yml`，点击`deploy`，若发现只有`Deploy`步骤报错，便配置成功

**此处有两个问题**

* `secrets.ACTIONS_DEPLOY_KEY`私钥和`xiaoming.github.io`的公钥未配置导致`Deploy`步骤报错
* 若未出现则无需理会。Actions会自动部署到`xiaoming.github.io`库的master分支，而现在`xiaoming.github.io`库可能默认为main分支，需要重命名main为master

## 生成并设置密钥

此处可参考：https://github.com/peaceiris/actions-gh-pages，Create SSH Deploy Key 处，有详细的位置图标

执行以下命令会生成两个文件

- `gh-pages.pub` 是公钥
- `gh-pages` 是私钥

```shell
ssh-keygen -t rsa -b 4096 -C "$(git config user.email)" -f gh-pages -N ""
```

* 公钥`gh-pages.pub`放在`xiaoming.github.io`中`Settings`的`Deploy keys`中，名字随意

* 私钥`gh-pages`放在`Blog_Private`中`Settings`的`Secrets`的`Actions`的`New repository secrets`，名字必须为`ACTIONS_DEPLOY_KEY`

配置完成后，再次点击`Actions`，点击中间`main.yml`，点击右边`Re-run all jobs`，无报错，便完成

## 自定义样式

{{< admonition info>}}

参考：https://lucas-0.github.io/

{{< /admonition>}}

创建文件：`\assets\css\\_custom.scss`

```scss
// ==============================
// Custom style
// 自定义样式
// ==============================
@import url('https://fonts.googleapis.com/css2?family=Rock+Salt&family=Noto+Serif+SC&family=Roboto+Slab:wght@100..900&display=swap');
.page {
    position: relative;
    max-width: 800px; //宽度限制800
    margin: 0 auto;
  }

```
创建文件：`\assets\css\\_override.scss`
```scss
// ==============================
// Override Variables
// 覆盖变量
// ==============================
// @import url('https://fonts.proxy.ustclug.org/css2?family=Rock+Salt&family=Noto+Serif+SC:wght@400;600;700&family=Roboto+Slab:wght@400;600;700&display=swap'); //使用中科大加速
// @import url('https://fonts.googleapis.com/css2?family=Rock+Salt&family=Noto+Serif+SC:wght@200..900&family=Roboto+Slab:wght@100..900&display=swap');

// Font and Line Height
$global-font-family: "Roboto Slab", "Noto Serif SC", "PingFang SC", "Hiragino Sans GB", "Microsoft Yahei", "WenQuanYi Micro Hei", "Segoe UI Emoji", "Segoe UI Symbol", Helvetica, Arial, -apple-system, system-ui, sans-serif;
$global-font-size: 18px;
$global-font-weight: 400; //粗细
$global-line-height: 1.75rem; //文本行的基线间的距离 原始1.5rem会让屏幕宽不够时h1重合
h1{font-size:1.5em;} //原始为2em，会与行高1.5rem冲突，修改为1.5em
$header-title-font-family: "Rock Salt", -apple-system, system-ui, sans-serif;
// Color of the secondary text
$global-font-secondary-color: #7d7d84;
```

## 集成不蒜子统计访问量

{{< admonition info>}}

参考：https://xwi88.com/hugo-plugin-busuanzi/

不蒜子：http://busuanzi.ibruce.info/

{{< /admonition>}}

将自定义配置添加在`[params.page]`文章页面全局配置里

```toml
 # xwi88 自定义配置 xwi88Cfg
[params.xwi88Cfg]
  [params.xwi88Cfg.summary]
    update = true # summary 更新日期显示
  [params.xwi88Cfg.page]
    update = true # pages 更新日期显示
  [params.xwi88Cfg.busuanzi]
    enable = true
    # custom uv for the whole site
    site_uv = true
    site_uv_pre = '<i class="fa fa-user"></i>' # 字符或提示语
    site_uv_post = ''
    # custom pv for the whole site
    site_pv = true
    site_pv_pre = '<i class="fa fa-eye"></i>'
    # site_pv_post = '<i class="far fa-eye fa-fw"></i>'
    site_pv_post = ''
    # custom pv span for one page only
    page_pv = true
    page_pv_pre = '<i class="far fa-eye fa-fw"></i>'
    page_pv_post = ''

```

在`layouts/partials/plugin`目录里创建文件`busuanzi.html`，如果不存在该目录就创建目录，文件内容如下：

```html
{{ if .params.enable }}
    {{ if eq .bsz_type "footer" }}
        {{/* 只有 footer 才刷新，防止页面进行多次调用，计数重复; 只要启用就计数，显示与否看具体设置 */}}
        <script async src="//busuanzi.ibruce.info/busuanzi/2.3/busuanzi.pure.mini.js"></script>
    {{ end }}

    {{ if or (eq .params.site_pv true) (eq .params.site_uv true) (eq .params.page_pv true) }}
        {{ if eq .bsz_type "footer" }}
            <section>
                {{ if eq .params.site_pv true }}
                    <span id="busuanzi_container_value_site_pv">
                        {{- with .params.page_pv_pre -}}
                            {{ . | safeHTML }}
                        {{ end }}
                        <span id="busuanzi_value_site_pv"></span>
                    </span>
                {{ end }}

                {{ if and (eq .params.site_pv true) (eq .params.site_uv true) }}
                    &nbsp;|&nbsp;              
                {{ end }}

                {{ if eq .params.site_uv true }}
                    <span id="busuanzi_container_value_site_uv">
                        {{- with .params.site_uv_pre -}}
                            {{ . | safeHTML }}
                        {{ end }}
                        <span id="busuanzi_value_site_uv"></span>
                    </span>
                {{ end }}
            </section>
        {{ end }}

        {{/*  page pv 只在 page 显示  */}}
        {{ if and (eq .params.page_pv true) (eq .bsz_type "page-reading") }}
            <span id="busuanzi_container_value_page_pv">
                {{- with .params.page_pv_pre -}}
                    {{ . | safeHTML }}
                {{ end }}
                <span id="busuanzi_value_page_pv"></span>&nbsp;
                {{- T "views" -}}
            </span>
        {{ end }}
    {{ end }}
{{ end }}

```

复制`themes/LoveIt/layouts/partials/footer.html`到`layouts/partials/footer.html`

在第9行`{{- end -}}`后回车添加：

```html
{{- /* busuanzi plugin */ -}}
{{- partial "plugin/busuanzi.html" (dict "params" .Site.Params.xwi88Cfg.busuanzi "bsz_type" "footer") -}}
```

复制`themes/LoveIt/layouts/posts/single.html`到`layouts/posts/single.html`

在第62行`{{- end -}}`后回车添加：

```html
{{- /* busuanzi plugin */ -}}
{{- partial "plugin/busuanzi.html" (dict "params" .Site.Params.xwi88Cfg.busuanzi "bsz_type" "page-reading") -}}
```

{{< admonition>}}

本地测试时出现大量的访问量是正常情况，部署到服务器上后，通过独自的url访问便会恢复正常

{{< /admonition>}}

