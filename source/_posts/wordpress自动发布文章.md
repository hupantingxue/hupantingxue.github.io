---
title: wordpress自动发布文章
date: 2020-11-05 21:08:35
tags:
- 创业
- 技术
- 自动化
categories: 创业
---
## 背景
最近上线了几个以wordpress搭建的内容型小程序，每天手动更新，既费事又费时，效率极其低下，于是计划用实现一套自动发布的程序。<!-- more -->

## 技术尝试
### xmlrpc
用wordpress的xmlrpc协议来发布文章，需要
1、 开启xmlrpc.php服务，wordpress 3.5以上版本自动支持；

2、 安装 wordpress_xmlrpc 插件：

> pip install python-wordpress-xmlrpc

3、 编写客户端：
```
#!/usr/bin/env python
# -*- coding: UTF-8 -*-
from wordpress_xmlrpc import Client, WordPressPost
from wordpress_xmlrpc.methods.posts import GetPosts, NewPost
from wordpress_xmlrpc.methods.users import GetUserInfo
from wordpress_xmlrpc.methods import posts
from wordpress_xmlrpc.methods import taxonomies
from wordpress_xmlrpc import WordPressTerm
from wordpress_xmlrpc.compat import xmlrpc_client
from wordpress_xmlrpc.methods import media, posts
import sys
import conf

reload(sys)
sys.setdefaultencoding('utf-8')

def main():
    connstr = "%s/xmlrpc.php" % (conf.WP_DOMAIN)
    wp = Client(connstr, conf.WP_ADMIN_USER, conf.WP_ADMIN_PWD)

    idx = 2
    post = WordPressPost()
    post.title = "十万个冷知识第%d集" % (idx)
    post.content = '''<video src="https://v.qq.com/x/cover/keupp5sxcs53vlc/j0173n2fwvd.html" object-fit="fill" controls="controls" width="250" height="180" data-mce-fragment="1"></video>'''
    post.post_status = 'publish'
    post.terms_names = {
        'post_tag': ['冷知识', '十万个冷知识'],
        'category': ['科学']
    }
    post.id = wp.call(posts.NewPost(post))

if "__main__" == __name__:
    main()
```
不知道是本地服务器问题，还是因为wordpress的xmlrpc协议开启失败，折腾了好几天，也没有成功，调用 xmlrpc validate 服务，一直返回 parse error。
在咨询了微慕的人之后，决定改用wordpress 的 REST API来发布文章。
### REST API

#### 1.安装 Application Passwords 插件
插件 -> 搜索“Application Passwords” -> 安装 -> 启用插件

#### 2.设置用户认证密码；
2.1 用户 -> 添加用户
2.2 搜索新创建的用户，将角色配置为“编辑”
2.3 编辑用户信息，添加"Application Passwords"的密码，并保存好，后续在发送http包头的时候需要用到；

#### 3. 配置好验证头

```
import base64

credentials = user + ':' + password
token = base64.b64encode(credentials.encode())
header = {'Authorization': 'Basic ' + token.decode('utf-8')}
```

#### 4. 发布一个简单的post

```
url = "https://domain/wp-json/wp/v2/"
post = {
         'title'    : vtitle,
         'status'   : 'publish',
         'content'  : content,
         'date'   : pubtime
        }
responce = requests.post(url + "/posts", headers=header, json=post)
```

#### 5. 发布一个带有feature image的post
```
def getImg(imgurl, vtitle):
    #save image
    imgfn = "images/%d%d.jpg" %(time.time(), random.randint(1000000, 9999999))
    urllib.urlretrieve(imgurl, imgfn)

    of = open(imgfn, 'rb')
    media = {
        'file': of,
        'caption': vtitle,
        'description': vtitle
    }
    r = requests.post(url + '/media', headers=header, files=media)
    #print "image resp: ", r.status_code, "\n", r.text
    rbody = json.loads(r.text)
    imgID = 0
    if 201 == r.status_code and "id" in rbody:
        imgID = rbody["id"]
    of.close()
    return imgID

post = {
        'title'    : vtitle,
        'status'   : 'publish',
        'content'  : content,
        'featured_media' : imgID,
        'date'   : pubtime
       }

responce = requests.post(url + "/posts", headers=header, json=post)
```

#### 6. 发布带有tag列表的post
tag发布的时候，需要携带name、slug、description、meta这4个参数，其中description、meta这2个参数可以为空。
slug需要注意的一点是，它必须是经过url编码之后的，否则会报错，可以调用 *urllib.quote()* 函数来编码，反之也可以用  *urllib.unquote()* 来解码；
以下是根据tag名称来创建tag并返回tagid的一个函数：

```
def getTags(taglist):
    tagids = []
    for tag in taglist:
        print tag
        slug = urllib.quote(tag.encode("utf-8"))
        tagjs = {"description":"", "name":tag.encode("utf-8"), "meta":[], "slug":slug}
        tagid = 0
        r = requests.post(url + "/tags", headers=header, json=tagjs)
        if 400 == r.status_code:
            rc = json.loads(r.text)
            tagid = rc["data"]["term_id"]
        elif 201 == r.status_code:
            rc = json.loads(r.text)
            tagid = rc["id"]
        else:
            pass
        if 0 != tagid:
            tagids.append(tagid)
    return tagids
post = {
         'title'    : vtitle,
         'status'   : 'publish',
         'content'  : content,
         'featured_media' : imgID,
         'tags' : tagIDs,
         #'categories': 5, category ID
         'date'   : pubtime
       }
```

#### 7. 中文分词
一般根据采集的标题来生成tag，这就需要用到中文分词，本文用的是python的jieba库：
```
import jieba

tags = jieba.cut_for_search(vtitle)
    taglist = []
    for tag in tags:
        if tag not in u'！，“<>《》？、_-':
            #print tag
            taglist.append(tag)
```

#### 8. 完整代码
```
#!/usr/bin/env python
#-*- coding:utf-8 -*-

import requests
import json
import base64
import time
import random
import urllib
#中文分词
import jieba
import re
import sys

url = conf.URL
user = conf.USER
password = conf.PASSWORD

credentials = user + ':' + password
token = base64.b64encode(credentials.encode())
header = {'Authorization': 'Basic ' + token.decode('utf-8')}

def getTags(taglist):
    tagids = []
    for tag in taglist:
        print tag
        slug = urllib.quote(tag.encode("utf-8"))
        tagjs = {"description":"", "name":tag.encode("utf-8"), "meta":[], "slug":slug}
        tagid = 0
        r = requests.post(url + "/tags", headers=header, json=tagjs)
        if 400 == r.status_code:
            rc = json.loads(r.text)
            tagid = rc["data"]["term_id"]
        elif 201 == r.status_code:
            rc = json.loads(r.text)
            tagid = rc["id"]
        else:
            pass
        if 0 != tagid:
            tagids.append(tagid)
    return tagids

def getImg(imgurl, vtitle):
    #save image
    imgfn = "images/%d%d.jpg" %(time.time(), random.randint(1000000, 9999999))
    urllib.urlretrieve(imgurl, imgfn)

    of = open(imgfn, 'rb')
    media = {
        'file': of,
        'caption': vtitle,
        'description': vtitle
    }
    r = requests.post(url + '/media', headers=header, files=media)
    #print "image resp: ", r.status_code, "\n", r.text
    rbody = json.loads(r.text)
    imgID = 0
    if 201 == r.status_code and "id" in rbody:
        imgID = rbody["id"]
    of.close()
    return imgID

def pubPost(imgurl, vtitle, vurl):
print "imgurl:%s|vtitle:%s|vurl:%s" %(imgurl, vtitle, vurl)
    imgID = 0

    if "" != imgurl:
        imgID = getImg(imgurl, vtitle)
        if 0 == imgID:
            print "image is null, then exit"
            return -1
        else:
            print "image id:", imgID

    pubtime = time.strftime("%Y-%m-%dT%H:%M:%S", time.localtime())
    # https://developer.wordpress.org/rest-api/reference/posts/
    content = '<video src="%s" object-fit="fill" controls="controls" width="250" height="180" data-mce-fragment="1"></video>' % vurl

    tags = jieba.cut_for_search(vtitle)
    taglist = []
    for tag in tags:
        if tag not in u'！，“<>《》？、_-':
            #print tag
            taglist.append(tag)

    tagIDs = getTags(taglist)
    print tagIDs
    #return -3

    if 0 != imgID:
        post = {
             'title'    : vtitle,
             'status'   : 'publish',
             'content'  : content,
             'featured_media' : imgID,
             'tags' : tagIDs,
             #'categories': 5, category ID
             'date'   : pubtime
            }
     else:
        post = {
             'title'    : vtitle,
             'status'   : 'publish',
             'content'  : content,
             'tags' : tagIDs,
             #'categories': 5, category ID
             'date'   : pubtime
            }
    responce = requests.post(url + "/posts", headers=header, json=post)
    print(responce.text)

def main(fn):
    with open(fn) as f:
        item = {}
        arr = []
        for line in f:
            if "href" in line:
                urls = re.findall(r"href=\"(.+?)html", line)
                if 0 < len(urls):
                    url = urls[0]
                else:
                    continue
                ss = "%shtml" %(url)
                item["url"] = ss
                item["pic"] = ""
            #if "episodeNumber" in line:
                titles = re.findall(r">(.+?)<", line)
                print titles
                if 1 < len(titles):
                    title = titles[1]
                else:
                    continue
                ss = "十万个冷知识第%s集" % title
                item["title"] = ss
            #if "pic" in item:
                arr.append(item)
                item = {}

        ii = 0
        for aa in arr:
            #print '<video src="%s" object-fit="fill" controls="controls" width="250" height="180" data-mce-fragment="1"></video>' % aa["url"]
            #print aa["url"]
            #print aa["pic"]
            #print aa
            vurl = aa["url"]
            vtitle = aa["title"]
            imgurl = aa["pic"]
            print "第%d次循环: url(%s)|title(%s)|img(%s)" % (ii, vurl, vtitle, imgurl)
            pubPost(imgurl, aa["title"], vurl)
            ii = ii + 1

if "__main__" == __name__:
    if (2 > len(sys.argv)):
        print "Usage: %s <txt>" % (sys.argv[0])
        sys.exit(-1)

    main(sys.argv[1])


```

#### 相关资料
[wordpress rest api官方文档](https://developer.wordpress.org/rest-api/)
[网友视频教程：用REST API发布POST](https://www.youtube.com/watch?v=AS6EPzDyjhA)
[Python中文分词工具大合集：安装、使用和测试](https://www.52nlp.cn/python%E4%B8%AD%E6%96%87%E5%88%86%E8%AF%8D%E5%B7%A5%E5%85%B7-%E5%90%88%E9%9B%86-%E5%88%86%E8%AF%8D%E5%AE%89%E8%A3%85-%E5%88%86%E8%AF%8D%E4%BD%BF%E7%94%A8-%E5%88%86%E8%AF%8D%E6%B5%8B%E8%AF%95)
