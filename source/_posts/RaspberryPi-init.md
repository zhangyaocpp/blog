---
title: 树莓派搭建私人服务器
date: 2017-01-23 01:37:49
categories: [树莓派]
tags: [树莓派,域名,DDNS]
---
阿里云服务器又涨价了，感觉已经负担不起了，但是又想拥有一台自己的私人服务器。某天，突然发现了树莓派，价格便宜、高可用。心血来潮！！说搞就搞！！

## 准备工作

1. 首先当然是有一台树莓派3代B型，淘宝价格￥190 + 周边 = ￥216 可以搞定（两个半月的阿里云ECS，还是蛮实惠的）
2. 支持端口映射的路由器（基本现在市面上的路由器都支持端口映射），我用的是小米路由器
3. 买一个属于自己的域名，如：www.uthinks.com
4. 具有公网ip的宽带，我家使用的联通20M宽带，很幸运有一个公网的IP。如果没有公网IP，需要借助花生壳来做内网穿透，不仅麻烦还有流量限制。
<!--more-->

## 树莓派装机
### 系统安装

1. 下载树莓派系统镜像（https://www.raspberrypi.org/downloads/）
	![image](http://www.uthinks.com:8081/resource/picture/Raspbian_image_download.png)

2. 接着就是把镜像烧入SD卡，windows下需要下载win32diskimager
   (http://download.csdn.net/detail/longerzone/7574047)。

3. 解压镜像和win32diskimager（绿色版打开即可使用），插入SD卡 --> 打开win32diskimager --> 添加镜像 --> 确认SD卡后点击烧写即可
	![image](http://www.uthinks.com:8081/resource/picture/win32diskimager.png)

4. 烧写结束后，在windows下SD卡会显示只有不到100M，不用担心，因为这个分区只是Linux 的boot分区，而其他内容作为Linux 的文件系统被挂载为ext4 文件系统，Windows 下识别不了而已

5. 把烧好的SD卡直接插入树莓派SD卡槽中，接上电源等待系统安装完成

### 系统配置
> 系统配置过程还是碰到很多坑，大家有什么问题可以直接联系我，我会补充出来

1. ssh无法登陆的问题
	由于树莓派默认没有打开sshd，所以我用HDMI连接上显示器，执行命令sudo raspi-config
	找到ssh然后enable后重启就ok了
	![image](http://www.uthinks.com:8081/resource/picture/sshd_init1.png)
	![image](http://www.uthinks.com:8081/resource/picture/sshd_init2.png)

## 动态域名解析（DDNS）

> 家里办理的联通宽带有公网IP，决定好好利用，但是公网IP不固定，需要动态修改域名解析。

### 注册域名

1. 在阿里云上购买自己喜欢的域名（抓紧时间备案，不然网站会被封）
	(https://wanwang.aliyun.com/domain/com?spm=5176.8142029.388261.128.anTrkC)

2. 如果有公网IP配置一条A记录，如果你使用的是花生壳配置一条CNAME记录
	![image](http://www.uthinks.com:8081/resource/picture/ddns1.png)

### 如何实现动态域名解析

> 下面给出的是python主要的核心代码，如果需要完整的环境代码请留言联系我。树莓派系统镜像中自带Python，还是很方便的

1. 获取自己的公网出口IP

		import urllib2

		def getIp():
    		try:
        		ip = visit("http://www.ip138.com/ip2city.asp")
    		except:
        		ip = "failed to get internet ip"
    		return ip

		def visit(url):
    		req = urllib2.Request(url)
    		opener = urllib2.urlopen(req)
    		result = opener.read()
    		return result[result.find('[') + 1: result.find(']')]

2. 下载alidns python SDK
(https://develop.aliyun.com/sdk/java?spm=5176.doc29772.416540.246.rjauTQ)

3. 解压安装
sudo python setup.py install

4. 安装alidns python SDK
pip install aliyun-python-sdk-alidns

5. 第1步获取到自己的公网IP后，调用API设置DNS解析

		import json

		from aliyunsdkalidns.request.v20150109 import UpdateDomainRecordRequest,DescribeDomainRecordsRequest, \
    		DescribeDomainRecordInfoRequest, AddDomainRecordRequest
		from aliyunsdkcore import client

		# 更新域名解析
		def updateDns(accessKey, accessKeySecret, hostRecord, dnsType, dnsValue, dnsRecordid, dnsTtl, returnFormat):
    		print hostRecord, dnsType, dnsValue, dnsRecordid, dnsTtl, returnFormat
    		clt = client.AcsClient(accessKey, accessKeySecret, 'cn-hangzhou')
    		request = UpdateDomainRecordRequest.UpdateDomainRecordRequest()
    		request.set_RR(hostRecord)
    		request.set_Type(dnsType)
    		request.set_Value(dnsValue)
    		request.set_RecordId(dnsRecordid)
    		request.set_TTL(dnsTtl)
    		request.set_accept_format(returnFormat)
    		result = clt.do_action(request)
    		return result

		# 获取当前的解析IP
		def getDnsIp(accessKey, accessKeySecret, dnsRecordid, returnFormat):
    		clt = client.AcsClient(accessKey, accessKeySecret, 'cn-hangzhou')
    		request = DescribeDomainRecordInfoRequest.DescribeDomainRecordInfoRequest()
    		request.set_accept_format(returnFormat)
    		request.set_RecordId(dnsRecordid)
    		result = clt.do_action(request)
    		result = json.JSONDecoder().decode(result)
    		result = result['Value']
    		return result

6. 路由器端口映射，配置完成记得点击保存并且生效
	![image](http://www.uthinks.com:8081/resource/picture/portDispatch.png)

7. 最后一步把动态解析脚本配置到crontab中定时执行

	*/1 * * * * /usr/bin/python /home/bill/basic/BasicTask.py

### 附：

1. accessKey、accessKeySecret如何获取
	登录阿里云控制台(https://ak-console.aliyun.com/#/accesskey)

2. 域名解析RecoreId如何获取

		# dns_domain 域名 如uthinks.com
		def check_records(dnsDomain):
    		clt = client.AcsClient(accessKeyId, accessKeySecret, 'cn-hangzhou')
    		request = DescribeDomainRecordsRequest.DescribeDomainRecordsRequest()
    		request.set_DomainName(dnsDomain)
    		request.set_accept_format('json')
    		result = clt.do_action(request)
    		print result
    		return result

    	返回值：
    	{
    		"PageNumber": 1,
    		"TotalCount": 2,
    		"PageSize": 20,
    		"RequestId": "***",
    		"DomainRecords": {
        		"Record": [
            		{
                		"RR": "*",
                		"Status": "ENABLE",
               		 	"Value": "****",
                		"RecordId": "****",
                		"Type": "A",
                		"DomainName": "uthinks.com",
                		"Locked": false,
                		"Line": "default",
                		"TTL": "600"
            		},
               ]
    		}
		}


# 如果我的文章对你有帮助，或者有什么疑问。欢迎在下方留言，一起交流讨论
