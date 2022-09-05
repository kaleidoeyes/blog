---
title: "使用Python爬取知乎日报并发送至邮箱"
date: 2022-05-16
hidden: false
draft: false
tags: ["Python"]
keywords: []
description: ""
slug: "py-zhihu"
---
<!--more-->
## 爬取内容

首先，导入需要的库：

```Python
import requests, yagmail, re
```

需要用到的Python库有requests, yagmail, re。其中requests和re用于爬虫，yagmail用于发送邮件。

```Python
url = 'https://news-at.zhihu.com/api/4/news/latest'
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36'
}
resp = requests.get(url, headers=headers)
```

url是我们要获取内容的链接，本文中即 https://news-at.zhihu.com/api/4/news/latest。

headers是一个防反爬的措施。如果把它去掉，平台的服务器会识别出你的这次请求是由自动化程序发出的，然后拒绝你的请求。你可以在requests中去除headers试一试。

```Python
print(resp.text)
```

不出意外的话，你可看到如下内容：

![](https://image.kaleidoeye.org/py-zhihu-1.jpg)

密密麻麻……分析一下，全部内容可分为两部分：stories和top_stories，之间有重复的部分，我们取后一个好了。

```Python
tops = resp.text.split("top_stories")[-1]
```

在我们截取的tops中，共有五篇内容（这也是我选择知乎日报的原因，因为并不需要沉浸于过多的信息中）。每篇有相同的格式：

-   image_hue
-   hint
-   url
-   image
-   title
-   ……

其中， 我们只需要image, url与title。

## 提取

下面编写正则表达式提取我们需要的内容：

```Python
obj = re.compile('"url":"(?P<url>.*?)","image":"(?P<image>.*?)","title":"(?P<title>.*?)","ga_prefix', re.S)
results = obj.findall(tops)
```

注意到，链接image与url中存在无用的“\”，稍后我们再整理。

## 邮件编写

邮件内容其实也是HTML，所以我们需要设计以下内容，当然你也可以不，但这样可以使内容更加美观。

```Python
contents = []
for result in results:
    url = result[0].replace('\\', '')  # 整理url
    image = result[1].replace('\\', '')  # 整理image
    title = result[2]
    contents.append(
        f'<a href="{url}"><p style="text-align: center;"><img src="{image}" width="216" height="144"'
        f' /></p></a>'
        f'<div align="center"><p style="font-size: 17px;font-weight: bold;">{title}</p>'
        f'<br/><hr/><br/>'
        f''
    )
```

contents是我们的内容容器，我们遍历results，将内容以相同的格式放入contents中。

## 邮件发送

### 配置yagmail信息

```Python
yag = yagmail.SMTP(user='###', password='###', host="smtp.###.com")
```

使用yagmail.SMTP来配置发送者的信息，user为发送者的邮箱。host中，如果发送者的邮箱为163邮箱，就填smtp.163.com；如果为qq邮箱就填smtp.qq.com。

注意，password中不是密码，而是邮箱的授权码！

如何获取授权密码？下面以163邮箱为例：

-   登录你的163邮箱；
-   来到设置，找到POP3/SMTP/IMAP;
-   在“开启服务”中把两个服务打开。为什么开启两个？因为我忘记是哪一个了……
-   在下方找到“授权密码管理”，点击“新增授权密码管理”。成功发送短信后会显示给你授权密码，复制即可。

## 发送邮件

```Python
yag.send(to="###", subject="知乎日报", contents=contents)
print("OK!")
```

to为接收者的邮箱，如果要发送多人，你可以采取列表的方式：[‘### ‘, ‘###’, ‘###’]。

subject为邮件的主题，contents即已编写好的邮件内容。

如此，邮件就已发送完毕，如果未接收到，可以到垃圾邮件中看看。

![](https://image.kaleidoeye.org/py-zhihu-2.jpg)

邮件演示

以下是全部代码：

```Python
import requests, yagmail, re

url = 'https://news-at.zhihu.com/api/4/news/latest'
headers = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/83.0.4103.61 Safari/537.36'
}
resp = requests.get(url, headers=headers)

tops = resp.text.split("top_stories")[-1]

contents = []
for result in results:
    url = result[0].replace('\\', '')
    image = result[1].replace('\\', '')
    title = result[2]
    contents.append(
        f'<a href="{url}"><p style="text-align: center;"><img src="{image}" width="216" height="144"'
        f' /></p></a>'
        f'<div align="center"><p style="font-size: 17px;font-weight: bold;">{title}</p>'
        f'<br/><hr/><br/>'
        f''
    )

yag = yagmail.SMTP(user='###', password='###', host="smtp.###.com")
yag.send(to="###", subject="知乎日报", contents=contents)
print("OK!")
```

结束。