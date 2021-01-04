---
layout: post
title: Hugo + Isso 搭建blog
tags: Linux Python Blog
---

[Hugo](https://gohugo.io/)是golang实现的静态网址生成器，主题丰富，使用简单。[Isso](https://posativ.org/isso/)则是python实现的类似Disqus的开源评论服务器。本站就是使用Hugo的一个极简主题[Even](https://themes.gohugo.io/hugo-theme-even/)生成静态内容，Isso server提供评论服务，Nginx作为web server反向代理Isso并提供静态内容。以下是我在centos7上的详细安装部署步骤：
   
### 安装Hugo
这里是按照源码方式安装的（官方文档有编译好的二进制下载安装方法说明），源码安装需要先安装golang（[安装文档点我](https://golang.org/doc/install)），按照文档安装完golang后，运行如下命令[安装Hugo](https://gohugo.io/getting-started/installing/)：
```bash
git clone https://github.com/gohugoio/hugo.git 
cd hugo 
go install --tags extended 
```

### 安装Even主题
##### 使用hugo创建站点
```bash
hugo new site myblog
```
##### 安装主题
```bash
cd myblog
git init 
git  submodule add https://github.com/olOwOlo/hugo-theme-even themes/even
```
##### 修改配置
可以把主题目录下的config.toml拷贝覆盖到hugo站点根目录myblog下的config.toml，然后根据自己的需求及环境做对应修改。每项配置都有详细注释。
##### 发表博客文章
hugo可以直接写Markdown格式的博文，构建时会将其转成网页。
```bash
hugo new post/my-first-post.md # 会根据模板生成一个Md文件（注意路径名是post不是posts），编辑此文件开始写文章
```
##### 调试站点效果
hugo本身带有一个web server，方便及时查看改动的网页效果。
```bash
hugo Server -D # 启动web server，打开浏览器输入 http://127.0.0.1:1313即可查看站点效果
# 几个有用的启动参数
hugo Server -D --bind "0.0.0.0" # --bind 指定不同ip绑定，默认是"127.0.0.1"
hugo Server -D --baseURL "http://127.0.0.1:1313" # --baseURL 指定站点的地址
```
##### 构建生成最终的静态网页
运行以下命令，会把站点所有静态内容文件生成在根目录下的public文件夹中，这也是后续正式部署的目标目录。
```bash
hugo -D
```

### 安装Isso
##### 安装python3
Isso需要python3支持，centos7上安装python3可以按如下方式：
```bash
# 安装第三方软件源EPEL
yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
# 安装python3
yum install python3
# 安装gcc等构建工具集
yum groupinstall 'Development Tools'
# 安装python C API headers等
yum install python3-devel.x86_64  
```

##### 安装python虚拟环境
```bash
pip3 install pipenv
mkdir isso
cd isso
pipenv --three # 在isso创建python3虚拟环境
```
pipenv会生成python3环境所需的所有文件到.venv文件夹，这个文件夹默认在~/.cache路径，可以设置export PIPENV_VENV_IN_PROJECT="true"环境变量，让其生成在工程目录。

##### 安装Isso
```bash
pipenv shell
pip install isso
```

##### 先根据自身需求修改Isso server配置
```ini
; |||||||||||| general settings ||||||||||||||
[general]
;database location. automatically created if doesn't already exist
dbpath = /home/yourname/isso/comments.db

;this will be the 'X-Script-Name' we proxy server requests to [later!]
name = isso

; your website or blog [not the location of Isso]
host =
    http://yoursite.com/
    https://yoursite.com/

; logfile. might need to create this first
log-file = /home/yourname/isso/isso.log

; get notifications by email
notify = smtp

;||||||||||| server section ||||||||||
[server]
;port to listen on. choose a number you like!
listen = http://localhost:8181/


;||||||||| smtp section [for notifications] |||||
[smtp]
;these are the details for the 'from' address Isso uses to send notifications
;you might want to use a dedicated email account for this
username = yourusername
password = yourpassword
host = smtp.yourmailhost.com
port = 587
security = starttls

;this is the 'to' address Isso sends notification emails to
to = you@youremail.com

;from address as shown on emails. should correspond to sender account above
from = "isso comments" <yourusername@yourmailhost.com>
timeout = 10

;|||||| guard –Isso's basic spam protection |||||||
[guard]
enabled = true
;no. of allowed comments per minute
ratelimit = 2
;no of direct replies allowed
direct-reply = 3
;can people reply to their own comments while edit window still open
reply-to-self = false
;do commenters need to leave a name
require-author = true
;do commenters need to provide an email
require-email = false

;allowed markdown in comments. [uses misaka markdown]
[markup]
;default options allow most 'unharmful' markdown
options = strikethrough, superscript, autolink
;default allowed = a, blockquote, br, code, del, em, h1, h2, h3, h4, h5, h6, hr, ins, li, ol, p, pre, strong, table, tbody, td, th, thead, ul
;allowed-elements =
;default allowed = align, href
;allowed-attributes

;creates identicons for users across isso installations
[hash]
;OK to use this salt
salt = Eech7co8Ohloopo9Ol6baimi
algorithm = pbkdf2

;comments must be moderated before publication
;NOTE: requires "notify = smtp" to be set in the [general] section
[moderation]
enabled=false
```
配置都有详细注释，也可见[文档描述](https://posativ.org/isso/docs/configuration/server/)。

##### 启动Isso
```bash
isso -c ~/isso/isso.cfg  
```
如果报错```ImportError: cannot import name SharedDataMiddleware```， 需要安装指定版本Werkzeug， 运行```pip install Werkzeug==0.16.1```。

##### Isso客户端配置
既在静态网页中需要引用Isso。修改hugo even主题文件：
myglob/themes/even/layouts/partials/comments.html，这个文件里面有各种comments系统的配置，把isso的客户端配置加在最后（{{- end }}之前）
```html
 {{ "<!-- isso -->" | safeHTML }}
  <script data-isso="{{ .Site.BaseURL }}isso/" data-isso-require-author="true"  src="{{ .Site.BaseURL }}isso/js/embed.min.js"></script>
  {{ "<!-- end isso -->" | safeHTML }}
  
  {{"<!-- begin comments //-->" | safeHTML}}
  <section id="isso-thread">
  </section>
  {{"<!-- end comments //-->" | safeHTML}}
```
更多配置见[文档](
https://posativ.org/isso/docs/configuration/client/)。

### 安装Ngix
```bash
yum install nginx
```

### 部署
##### isso部署
```bash
#添加新用户和组
useradd -m -r -s /sbin/nologin isso
#添加工作目录
chown -R isso:isso /home/isso
```
##### 配置isso service
新建文件/etc/systemd/system/isso.service
```conf
[Unit]
Description=Isso Comments
#start after networking stuff has loaded
After=network.target

[Service]
#your name
User=isso
#group webserver runs under when serving web pages
Group=isso
WorkingDirectory=/home/isso
#max file dscriptor [equivalent of ulimit in terminal]
LimitNOFILE=8192
#location of isso script to run & location of its config file
#note the isso script is in the `bin` folder in your virtual environment
#you don't need to activate your virtual environment here
#it will automatically be activated when the isso script within it runs
ExecStart=/home/isso/.venv/bin/isso  -c /home/isso/isso.cfg
#respawn isso process if it crashes
Restart=on-failure
#wait 10 mins to respawn, if it's crashing constantly
StartLimitInterval=600

[Install]
#target [??? not sure what this means]
WantedBy=multi-user.target

#don't forget to enable it: sudo systemctl enable isso
```
设置开机启动
```bash
systemctl enable isso
```
##### 部署Nginx
创建静态网页目录/www/data/
```bash
mkdir -p  /www/data/myblog/public
chown -R nginx:nginx /www/data
```
nginx配置如下
```conf
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location /isso {
            proxy_set_header X-Script-Name /isso;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_pass http://127.0.0.1:8181;
        }

        location / {
            root /www/data/myblog/public;
        }
   }
   }
```
把hugo生成的public目录上传到服务器的/www/data/myblog/public位置
```bash
rsync -avz --delete public/ www-data@ftp.topologix.fr:~/www/
```
##### 启动服务
```bash
systemctl start nginx.service
systemctl start isso.service
```


### 参考
 [Hugo Documentation](https://gohugo.io/documentation/)

[Isso Documentation](https://posativ.org/isso/docs/)

[Hugo with Isso](https://stiobhart.net/2017-02-24-isso-comments/) 
