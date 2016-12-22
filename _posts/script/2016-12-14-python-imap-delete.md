---
layout: post
title:  "腾讯企业邮箱邮件用python批量删除"
date:   2016-12-14 12:50:00 +0800
categories: "工具脚本"
tag: ["python"]
---

* content
{:toc}



企业邮箱太多邮件删不清楚了,foxmail收取邮件以后一次性清理掉

代码[参考地址](http://codego.net/9094239/)

执行前线先通过页面关闭微信登陆认证

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import imaplib
import time

# 拆分删除列表,一次性标记删除1000个
def chunks(l, n):
    # yields successive n-sized chunks from l.
    for i in xrange(0, len(l), n):
        yield l[i:i+n]

def main():
        conn = imaplib.IMAP4_SSL('imap.exmail.qq.com', 993)
        imaplib.Debug = 1
        conn.login("name@domain.com", "password")
        print conn.welcome
        # 列出邮箱中的文件夹
        # box_list_info_buffer = conn.list()
        # box_list = box_list_info_buffer[1]
        # # print type(box_list)
        # 默认邮箱文件夹
        box_name = 'INBOX'
        # 选择邮箱
        conn.select(box_name)
        # 获取所有邮件的id
        ttype, data = conn.search(None, 'ALL')
        data_list =  data[0].split()
        # 将所有邮件标记为删除
        for i in list(chunks(data_list, 1000)):
            ret = conn.store(",".join(i), '+FLAGS', '\\Deleted')
            # print ret
        print 'commit dlete'
        _start = time.time()
        conn.expunge()
        print time.time() - _start
        conn.logout()

if __name__=="__main__":
        main()
```

7000封邮件大概执行了3-5分钟,脚本停止后,大概有3-5分钟邮箱网页不能登陆
