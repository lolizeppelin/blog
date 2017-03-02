---
layout: post
title:  "linux账户密码生成,批量生成账户密码"
date:   2012-06-27 12:50:00 +0800
categories: "工具脚本"
tag: ["linux"]
---

* content
{:toc}

用得少,但是要的时候总是忘记,记录起来

批量生成账户

```shell
useradd -u 1000 -g 1000 -G dba -p `openssl passwd -1 -salt "yoursalt" "yourpass"` new_user
```


root修改指定账户密码

```shell
#!/bin/bash
echo "yourpass" | passwd root --stdin
```
