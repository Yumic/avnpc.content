---
published: true
date: '2015-01-21 01:55:48'
tags:
  - php
  - nginx
  - osx
  - homebrew
  - mysql
author: AlloVince
title: Mac 下安装 LNMP(Nginx+PHP5.6)环境
---

## 安装 Homebrew

最近工作环境切换到 Mac，所以以 OS X Yosemite（10.10.1）为例，记录一下[从零开始安装 Mac 下 LNMP 环境的过程](http://avnpc.com/pages/install-lnmp-on-osx)

确保系统已经安装 xcode，然后使用一行命令安装依赖管理工具[Homebrew](http://brew.sh/)

```bash
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

之后就可以使用

```bash
brew install FORMULA
```
   
来安装所需要的依赖了。

brew（意为酿酒）的命名很有意思，全部都使用了酿酒过程中采用的材料/器具，名词对应以下的概念：

- Formula（配方） 程序包定义，本质上是一个 rb 文件
- Keg（桶）程序包的安装路径
- Cellar（地窖）所有程序包（桶）的根目录
- Tap（水龙头）程序包的源
- Bottle （瓶子）编译打包好的程序包

最终编译安装完毕的程序就是一桶酿造好的酒

更详细的信息参考[Homebrew 的官方 Cookbook](https://github.com/Homebrew/homebrew/blob/master/share/doc/homebrew/Formula-Cookbook.md)

因此使用 Homebrew 常见的流程是：

1. 增加一个程序源（新增一个水龙头） `brew tap homebrew/php`
2. 更新程序源 `brew update`
3. 安装程序包（按照配方酿酒） `brew install git`
4. 查看配置 `brew config` 可以看到程序包默认安装在`/usr/local/Cellar`下 （酒桶放在地窖内）

## 安装 PHP5.6（FPM 方式）

首先加入 Homebrew 官方的几个软件源


```bash
brew tap homebrew/dupes
brew tap homebrew/versions
brew tap homebrew/php
```

PHP 如果采用默认配置安装，会编译`mod_php`模块并只运行在 Apache 环境下，为了使用 Nginx，这里需要编译 php-fpm 并且禁用 apache，主要通过参数`--without-fpm --without-apache`来实现。完整的安装指令为


```bash
brew install php56 \
--without-snmp \
--without-apache \
--with-debug \
--with-fpm \
--with-intl \
--with-homebrew-curl \
--with-homebrew-libxslt \
--with-homebrew-openssl \
--with-imap \
--with-mysql \
--with-tidy
```

由于 OSX 已经自带了 PHP 环境，因此需要修改系统路径，优先运行 brew 安装的版本，在`~/.bashrc`里加入：

```bash
export PATH="/usr/local/bin:/usr/local/sbin:$PATH"
```

如果要安装新的 php 扩展，可以直接安装而不用每次重新编译 php，所有的扩展可以通过

```bash
brew search php56
```
    
看到，下面是我自己所需要的扩展，可以支持[Phalcon 框架](http://phalconphp.com/)：

```bash
brew install php56-gearman php56-msgpack php56-memcache php56-memcached php56-mongo  php56-phalcon php56-redis php56-xdebug
```
    
### PHP-FPM 的加载与启动

安装完毕后可以通过以下指令启动和停止 php-fpm

```bash
php-fpm -D
killall php-fpm
```

同时可以将 php-fpm 加入开机启动

```bash
ln -sfv /usr/local/opt/php56/*.plist ~/Library/LaunchAgents
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.php56.plist
```


## 安装 Nginx


```bash
brew install nginx
```
    
安装完毕后可以通过

```bash
nginx
nginx -s quit
```
    
启动和关闭，同时也支持重载配置文件等操作

```bash
nginx -s reload|reopen|stop|quit
```

nginx 安装后默认监听 8080 端口，可以访问`http://localhost:8080`查看状态。如果要想监听 80 端口需要 root 权限，运行

```bash
sudo chown root:wheel /usr/local/Cellar/nginx/1.6.2/bin/nginx
sudo chmod u+s /usr/local/Cellar/nginx/1.6.2/bin/nginx
```
    
并使用 root 权限启动

```bash
sudo nginx
```

开机启动

```bash
ln -sfv /usr/local/opt/nginx/*.plist ~/Library/LaunchAgents
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist
```
    
### Nginx + PHP-FPM 配置

Nginx 一般都会运行多个域名，因此这里参考了[@fish 的方法](http://segmentfault.com/blog/fish/1190000000606752)，按 Ubuntu 的文件夹结构来存放 Nginx 的配置文件

```bash
mkdir -p /usr/local/var/logs/nginx
mkdir -p /usr/local/etc/nginx/sites-available
mkdir -p /usr/local/etc/nginx/sites-enabled
mkdir -p /usr/local/etc/nginx/conf.d
mkdir -p /usr/local/etc/nginx/ssl
```

编辑 Nginx 全局配置

```bash
vim /usr/local/etc/nginx/nginx.conf
```

```bash
worker_processes  1;
error_log   /usr/local/var/logs/nginx/error.log debug;
pid        /usr/local/var/run/nginx.pid;

events {
    worker_connections  256;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] '
	    '"$request" $status $body_bytes_sent '
	    '"$http_referer" "$http_user_agent" '
	    '"$http_x_forwarded_for" $host $request_time $upstream_response_time $scheme '
	    '$cookie_evalogin';

    access_log  /usr/local/var/logs/access.log  main;

    sendfile        on;
    keepalive_timeout  65;
    port_in_redirect off;

    include /usr/local/etc/nginx/sites-enabled/*;
}
```


这样一来首先可以把一些可复用配置独立出来放在`/usr/local/etc/nginx/conf.d`下，比如 fastcgi 的设置就可以独立出来

```bash
vim /usr/local/etc/nginx/conf.d/php-fpm
```
    
内容为

```bash
location ~ \.php$ {
    try_files                   $uri = 404;
    fastcgi_pass                127.0.0.1:9000;
    fastcgi_index               index.php;
    fastcgi_intercept_errors    on;
    include /usr/local/etc/nginx/fastcgi.conf;
}
```


然后`/usr/local/etc/nginx/sites-enabled`目录下可以一个文件对应一个域名的配置，比如 web 服务器目录是`/opt/htdocs`

```bash
vim /usr/local/etc/nginx/sites-enabled/default
```

```bash
server {
    listen       80;
    server_name  localhost;
    root         /opt/htdocs/;

    location / {
        index  index.html index.htm index.php;
        include     /usr/local/etc/nginx/conf.d/php-fpm;
    }
}
```

此时启动了 php-fpm 并且启动了 Nginx 后，就可以通过`http://localhost`来运行 php 程序了

## 安装 MySQL

```bash
brew install mysql
```
    
可以通过

```bash
mysql.server start
mysql.server stop
```
    
来启动／停止，启动后默认应为空密码，可以通过 mysqladmin 设置一个密码

```bash
mysqladmin -uroot password "mypassword"
```

但是在操作的时候出现了空密码无法登入的情况，最终只能通过 mysqld_safe 来设置

```bash
sudo mysqld_safe --skip-grant-tables
mysql -u root
mysql> UPDATE mysql.user SET Password=PASSWORD('mypassword') WHERE User='root';
mysql> FLUSH PRIVILEGES;
```


最后将 MySQL 加入开机启动

```bash
cp /usr/local/Cellar/mysql/5.6.22/homebrew.mxcl.mysql.plist ~/Library/LaunchAgents/
```
    
## Memcache

```bash
brew install memcached
```
    
启动/停止指令

```bash
memcached -d
killall memcached
```
    
加入开机启动

```bash
cp /usr/local/Cellar/memcached/1.4.20/homebrew.mxcl.memcached.plist ~/Library/LaunchAgents/
```
   
## Redis 
    
```bash
brew install redis
```
    
Redis 默认配置文件不允许以 Deamon 方式运行，因此需要先修改配置文件

```bash
vim /usr/local/etc/redis.conf
```
    
将 daemonize 修改为 yes，然后载入配置文件即可实现后台进程启动

```bash
redis-server /usr/local/etc/redis.conf
```

加入开机启动   
   
```bash
cp /usr/local/Cellar/redis/2.8.19/homebrew.mxcl.redis.plist ~/Library/LaunchAgents/ 
```

## 设置别名

最后可以对所有服务的启动停止设置别名方便操作


```bash
vim ~/.bash_profile
```
    
加入

```bash
alias nginx.start='launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist'
alias nginx.stop='launchctl unload -w ~/Library/LaunchAgents/homebrew.mxcl.nginx.plist'
alias nginx.restart='nginx.stop && nginx.start'
alias php-fpm.start="launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.php55.plist"
alias php-fpm.stop="launchctl unload -w ~/Library/LaunchAgents/homebrew.mxcl.php55.plist"
alias php-fpm.restart='php-fpm.stop && php-fpm.start'
alias mysql.start="launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist"
alias mysql.stop="launchctl unload -w ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist"
alias mysql.restart='mysql.stop && mysql.start'
alias redis.start="launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.redis.plist"
alias redis.stop="launchctl unload -w ~/Library/LaunchAgents/homebrew.mxcl.redis.plist"
alias redis.restart='redis.stop && redis.start'
alias memcached.start="launchctl load -w ~/Library/LaunchAgents/homebrew.mxcl.memcached.plist"
alias memcached.stop="launchctl unload -w ~/Library/LaunchAgents/homebrew.mxcl.memcached.plist"
alias memcached.restart='memcached.stop && memcached.start'
```

## 安装其他项目支持


```bash
brew install composer node
```
    
## 安装 Oh My Zsh

```bash
brew install zsh-completions
chsh -s /usr/local/bin/zsh
vim ~/.zshenv
```

加入内容

```bash
export PATH=/usr/local/bin:$PATH
```

然后
    
```bash
vim ~/.zshrc
```
    
加入内容

```bash
fpath=(/usr/local/share/zsh-completions $fpath)
autoload -Uz compinit
compinit -u
```
    
最后运行

```bash
rm -f ~/.zcompdump; compinit
```

查看正在使用的 shell

```bash
dscl localhost -read Local/Default/Users/$USER UserShell
```
    
安装 Oh My Zsh

```bash
wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | sh
```
    
## 参考

- [全新安装 Mac OSX 开发者环境](http://segmentfault.com/blog/fish/1190000000606752)



