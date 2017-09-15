---
title: 基于Hexo + Git + Nginx的博客发布
date: 2017-01-23 22:16:00
categories: [Hexo建站]
tags: [后端,Hexo,Git,Nginx]
---

进过上一篇《树莓派搭建私人服务器》,我们已经有一个私人服务器了，现在需要做点什么实际事情了，先搭一个博客分享自己的经验吧。

相关文章：
1.《树莓派搭建私人服务器》
	(http://www.uthinks.xyz/2017/01/23/RaspberryPi-init/)

## 准备工作
1. 环境已经初始化的树莓派
2. Git服务器，我用的是自己搭建的Git服务器，当然也可以使用GitHud
3. Nginx
4. Hexo，我朋友已经写过关于Hexo详细的文档，这里就不在赘述。
	(http://luckykun.com/work/2016-04-23/heoll-hexo.html)

<!--more-->
## Git服务器搭建

1. 首先在树莓派上安装Git，同时确保ssh已经正确安装并且默认开启

		sudo apt-get install wget git-core

2. 添加git用户和组，其实就是Linux普通用户就行

		adduser git
		passwd git

3. 切换到git用户，增加一个新的Git仓库

		cd /home/git
		mkdir blog
		cd blog
		git init --bare

4. 本地把blog项目迁移下来, 刚刚初始化的仓库是没有任何分支的，后面提交代码的时候会自动生成一个master分支。后面hexo直接在该目录下搭建

		git clone git@[域名 | IP]:/home/git/blog

5. 至此，一个自己的git服务器已经搭建完成

### 附录

1. 初始化空Git仓库git init 和 git init --bare有什么区别
	(http://blog.csdn.net/feizxiang3/article/details/8065506)

## Nginx安装配置

1. 首先安装nginx

		sudo apt-get install nginx

2. 修改nginx配置文件 /etc/nginx/nginx.conf

		user www-data;
		worker_processes 4;
		pid /run/nginx.pid;

		events {
			worker_connections 768;
			# multi_accept on;
		}

		http {
				server {
			      		listen  8081;
			      	    # 这个目录是hexo生成的页面路径，后面提交到git的页面同步到这个目录下
					  	location / {
			          			root   /home/bill/blog/public;
								index  index.html;
			      		}

			      		# 这个配置是静态文件：图片、文件下载路径，有另外一个git仓库管理，一样同步到对应目录下
				  		location ~ ^/resource/picture/(.*)? {
			         		alias /home/bill/resource/picture/$1 ;
			          		add_header Cache-control max-age=7200,s-maxage=3600;
			      		}
	      		}

				##
				# Basic Settings
				##

				sendfile on;
				tcp_nopush on;
				tcp_nodelay on;
				keepalive_timeout 65;
				types_hash_max_size 2048;
				# server_tokens off;

				# server_names_hash_bucket_size 64;
				# server_name_in_redirect off;

				include /etc/nginx/mime.types;
				default_type application/octet-stream;

				##
				# SSL Settings
				##

				ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
				ssl_prefer_server_ciphers on;

				##
				# Logging Settings
				##

				access_log /var/log/nginx/access.log;
				error_log /var/log/nginx/error.log;

				##
				# Gzip Settings
				##

				gzip on;
				gzip_disable "msie6";

				# gzip_vary on;
				# gzip_proxied any;
				# gzip_comp_level 6;
				# gzip_buffers 16 8k;
				# gzip_http_version 1.1;
				# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

				##
				# Virtual Host Configs
				##

				include /etc/nginx/conf.d/*.conf;
				include /etc/nginx/sites-enabled/*;
		}

3. 启动nginx

		sudo /etc/init.d/nginx start | restart | stop

## 博客的发布

环境已经搭建完成了，如何实现把文件提交到Git自动同步到nginx目录下完成发布呢？

1.如果你使用的GitHud的话这个问题很简单，这里就不在赘述。参考我朋友的hexo教程
	(http://luckykun.com/work/2016-04-23/heoll-hexo.html)

2.现在说说我的方法，其实也很简单
* 编写一个简单的git pull脚本

	import os

	if __name__ == '__main__':
		#blog分支同步
    	os.chdir('/home/bill/blog/')
    	os.system('git pull origin master')

		#静态文件同步
    	os.chdir('/home/bill/resource/')
    	os.system('git pull origin master')

* 定时任务配置 crontab -e

		*/1 * * * * /usr/bin/python /home/bill/deploy/deploy.py


# 如果我的文章对你有帮助，或者有什么疑问。欢迎在下方留言，一起交流讨论
