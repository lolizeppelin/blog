---
layout: post
title:  "python编辑大文件,自动下载ucloud备份的数据库文件并导入数据库"
date:   2016-12-14 12:50:00 +0800
categories: "工具脚本"
tag: ["python"]
---

* content
{:toc}


python编辑大文思路。

    先通过f.open对象readline找到需要编辑开始和结束的字节数。
    再通过mmap加偏移量映射这部分文件然后写回。


ucloud的登录图片比较简单，还是能实现自动登录的

    由于ucloud的数据库是备份全库的，所以需要编辑ucloud的数据库文件
    将mysql库删除再倒入本地数据库，这就需要编辑大文件并修改，于是就有了上面的python编辑大文件。
    导入的时候，如果只需要做一些数据查询
    可以启动一个只有MyISAM引擎的数据库,这样导入速度较快(数据库会忽略指定表的存储引擎)


只启用MyISAM的配置如下

    skip-innodb
    default-storage-engine = MyISAM
    innodb=OFF

下面下载ucloud备份数据库并处理的代码（部分参数和url与ucloud机房有关，比如1001是ucloud A机房）

    需要安装pyocr


```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import urllib
import urllib2
import cookielib
import shutil
import os
import sys
import re
import subprocess
import pyocr
import json
import Image
import time
#import pdb
import mmap
import pwd

def open_mainpage():
    #打开登录页面,初始化opener
    url_main_page = 'https://account.ucloud.cn/cas/login'
    cj = cookielib.LWPCookieJar()
    opener = urllib2.build_opener(urllib2.HTTPCookieProcessor(cj))
    urllib2.install_opener(opener)
    res = urllib2.Request(url_main_page)
    req = opener.open(res)
    return opener, req.read()


def get_lt_code(main_pages):
    #从登录页面获取LT code，这是登录时需要post的一个value
    filter_lt = re.compile('LT-[A-Za-z0-9]{20,40}')
    page_text = main_pages.strip()
    if filter_lt.search(page_text,re.X) is not None:
        return filter_lt.search(page_text).group(0)
    else:
        #print main_pages
        return ''


def get_verification_code(opener, verification_code_file = 'D:\\Downloads\\1.png'):
    #获取验证码文件，并用pyocr解析验证码
    url_verification_code = 'https://account.ucloud.cn/cas/login/verification_code'
    res = urllib2.Request(url_verification_code)
    req = opener.open(res)

    try:
        if os.path.isfile(verification_code_file):os.remove(verification_code_file)
        with open(verification_code_file, 'wb') as fp:
            shutil.copyfileobj(req, fp)
    except:
        return None
    tools = pyocr.get_available_tools()[:]
    if len(tools) == 0:
        print("No OCR tool found")
        return None
    print("Using [%s] to get key from image" % (tools[0].get_name()))
    verification_code = tools[1].image_to_string(Image.open(verification_code_file))
    return str(verification_code)


def post_login(opener, post_data):
    url_login = 'https://account.ucloud.cn/cas/login'
    for i in range(5):
        post_data['verify_code'] = get_verification_code(opener, '/tmp/1.png')
        if post_data['verify_code'] is None:
            continue
        res = urllib2.Request(url_login, urllib.urlencode(post_data))
        req = opener.open(res)
        json_response = json.loads(req.read())
        if json_response['ret_code'] == 0:
            return True
        else:
            print json_response['ret_code']
            #print json_response['error_message']
            if i == 4:
                return False
            continue


def page_db(opener):
    #切换到dbbackup页面
    url_db = 'https://udb.ucloud.cn/udb/backup'
    headers = {'User-Agent' : 'Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.2; Win64; x64; Trident/6.0)'}
    res = urllib2.Request(url_db, headers = headers)
    req = opener.open(res)
    #print req.read()


def get_db_list(opener=None, db_name_list=None):
    #获取db列表
    if db_name_list is None:
        return None
    db_list = None
    #先查询本地db列表文件，如果存在不再去ucloud获取db列表
    try:
        f = open('./db.list')
        db_list_text = f.readlines()
        f.close()
        db_list = json.loads(db_list_text[0].strip())['data']
    except Exception,e:
        print 'error',e
        return None

    #不存在本地db列表，去ucloud获取
    if db_list is None or type(db_list) != type([]):
        list_url = 'https://udb.ucloud.cn/api/udb/instances?db_kind=2&max_count=9999&offset=0&use_session=yes&format=json®ion_id=1001&zone_id=1&_=%d' % int(time.time()*1000)
        print 'list not ok! get url [%s]' % list_url
        try:
            #pdb.set_trace()
            res = urllib2.Request(list_url)
            req = opener.open(res)
            #print 'req is ',req.read().strip()
            json_text = req.read()
            db_list = json.loads(json_text.strip())['data']
            try:
                f = open('./db.list','w')
                f.writelines(json_text)
                f.close()
            except Exception, e:
                print 'wtfffff',e
                pass
        except Exception,e:
            print 'error open db_list url', e

    if db_list is None or type(db_list) != type([]):
        return None

    db_dict = {}
    for db in db_list:
        if db['instance_name'] in db_name_list:
            db_dict[db['instance_name']] = {}
            db_dict[db['instance_name']]['db_id'] = db['db_id']
            db_dict[db['instance_name']]['ip'] = db['virtual_ip']
            db_dict[db['instance_name']]['port'] = db['port']

    return db_dict


def get_down_url(opener, db_dict):
    #获取db文件下载地址
    for instance_name in db_dict.keys():
        api_get_download_id = 'https://udb.ucloud.cn/api/udb/backup?offset=0&max_count=9999&db_id=%s&use_session=yes&format=json®ion_id=1001&zone_id=1&_=%d' % (db_dict[instance_name]['db_id'],int(time.time()*1000))
        api_get_download = 'https://udb.ucloud.cn/udb/create_url'
        #print api_get_download_id
        back_list = None
        try:
            #pdb.set_trace()
            res = urllib2.Request(api_get_download_id)
            req = opener.open(res)
            #print 'req is ',req.read().strip()
            json_text = req.read()
            back_list = json.loads(json_text.strip())['data']
        except Exception, e:
            print 'error open get download api',e

        if back_list is None:
            return None

        post_data = {'backup_id':0, 'use_session':'yes', 'format':'json', 'region_id':1001, 'zone_id':1}
        #print back_list[0],back_list[0].keys()
        if back_list[0]['dbid'] == db_dict[instance_name]['db_id']:
            post_data['backup_id'] = back_list[0]['id']
            #pdb.set_trace()
            headers = {'User-Agent' : 'Mozilla/5.0 (compatible; MSIE 10.0; Windows NT 6.2; Win64; x64; Trident/6.0)',
                       'X-Requested-With': 'XMLHttpRequest',
                       'Referer': 'https://udb.ucloud.cn/udb/backup',
                       'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'}
            res = urllib2.Request(api_get_download, urllib.urlencode(post_data), headers = headers)
            req = opener.open(res)
            down_md5 = req.read()
            #print len(down_md5)
            if len(down_md5) == 40:
                down_url = 'http://data.1001.ucloud.cn/download/index/1001/%s' % down_md5
                db_dict[instance_name]['down_url'] = down_url


def down_file_by_wget(db_dict):
    #使用wget下载db文件
    try:
        shutil.rmtree('./tmp')
    except:
        pass
    time.sleep(5)
    for instance_name in db_dict.keys():
        if 'down_url' in db_dict[instance_name].keys():
            command = '/usr/bin/wget -q %s -O ./%s.tgz' % (db_dict[instance_name]['down_url'], instance_name)
            down_process = subprocess.Popen(command,shell=True)
            down_process.wait()
            #print 'finsh download file %s,start untar' % instance_name
            command = '/bin/tar -xf %s.tgz -C ./' % (instance_name)
            #print command
            untar_process = subprocess.Popen(command,shell=True)
            untar_process.wait()
            #print 'untar finsh'


def import_db(db_dict, max_serach_num=800,max_mysqldb_len=30):
    #将sql文件编辑修改后倒入本地数据库
    page_size = 4096
    return_value = {'code':0, 'reason':'import database success'}
    sql_files = os.listdir('./tmp/')
    if len(sql_files) != len(db_dict.keys()):
        print 'error'
    for file_name in sql_files:
        full_file_path = './tmp/' + file_name
        # delte system database mysql from sql file
        f = open(full_file_path,'r+')
        count_len = 0
        start_len = 0
        end_len = 0
        for i in range(max_serach_num):
            line = f.readline()
            count_len += len(line)
            #获取创建mysql数据库的起始位置
            if line[0:28] == "-- Current Database: `mysql`":
                print 'fine mysql len is ' + str(count_len)
                start_len = count_len
            #获取创建mysql数据库的结束位置
            if line[0:40] == "-- Dumping routines for database 'mysql'":
                end_len = count_len
                break
            if i == max_serach_num - 1:
                return_value['code'] = 1
                return_value['reason'] = 'find mysql databse start or end line fail over max'
                return return_value
        if not end_len > start_len:
                return_value['code'] = 1
                return_value['reason'] = 'find mysql databse start or end line fail'
                return return_value

        #偏移量必须是pagesize的整数倍，补全下偏移量
        completion_pagesize = start_len % page_size
        print 'cp size is %d ' % completion_pagesize
        start_len = start_len - completion_pagesize
        mmap_len = end_len - start_len
        #映射数据库文件 创建mysql的文件断
        map = mmap.mmap(f.fileno(), length=mmap_len, access = mmap.ACCESS_WRITE, offset=start_len)

        mark_start = 0
        mark_end = 0
        for i in range(max_mysqldb_len):
            mark = map.tell()
            line = map.readline()
            print line
            if line[0:28] == "-- Current Database: `mysql`":
                #创建位置标记
                mark_start = mark
            if line[0:40] == "-- Dumping routines for database 'mysql'":
                #结束位置标记
                mark_end = map.tell()
                break
            if i == max_mysqldb_len - 1 :
                return_value['code'] = 1
                return_value['reason'] = 'mysql database too large!'

        #头3字节转为注释符号
        map[mark_start:mark_start+3] = '/*\n'
        #用@填充
        for k in range(mark_start+3, mark_end):
            map[k] = '@'
        #尾3字节添加注释符号
        map[-3:mark_end] = '*/\n'
        map.flush()
        f.close()

        print 'delete databae mysql.mysql finsh!!'
        # change innodb to myisam
        command = "/bin/sed -i s/^\)\ ENGINE=InnoDB/\)\ ENGINE=MyISAM/ %s" % full_file_path
        print command
        sed_process = subprocess.Popen(command,shell=True)
        sed_process.wait()
        # import sql file
        command = '/usr/bin/mysql -uroot -S /var/lib/mysql/mysql_tmp.sock < %s' % full_file_path
        import_process = subprocess.Popen(command,shell=True)
        import_process.wait()
        return return_value



def main():
    curuid = os.getuid()
    user_info = pwd.getpwuid(curuid)
    home_dir = user_info[5]
    user_name = user_info[0]
    crontab_dir = home_dir + '/crontab_shell/'
    try:
        os.chdir(crontab_dir)
    # print os.getcwd()
    except:
        return None
    #return None
    post_data = {}
    post_data['username'] = '帐号'
    post_data['password'] = '密码'
    post_data['verify_code'] = ''
    post_data['lt'] = ''
    post_data['service'] = ''

    global_openr = None
    global_openr, main_pages = open_mainpage()
    post_data['lt'] = get_lt_code(main_pages)
    if not post_login(global_openr, post_data):
        print 'get login faile'
        sys.exit(1)
    page_db(global_openr)
    db_name_list = ('your db 1','your db 2')
    db_dict = get_db_list(global_openr, db_name_list)
    get_down_url(global_openr, db_dict)
    print 'down load db file from ucloud'
    down_file_by_wget(db_dict)
    print 'download finsh start import db file'
    print import_db(db_dict, 5000, 5000)


if __name__ == '__main__':
    main()
```

一个简易的去除sql文件开头的建表语句的示例

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import re
# drop与create table的正则
re_intance = re.compile('(drop table.+?;\n|create table[\s\S]+?;\n)', re.IGNORECASE)
buffer_size = 4096


def replace_function(input):
    # re.sub的替换对象可以是一个函数
    # 这个函数收到的参数是re match对象
    # 我们将match对象的具体内容替换为空字符串
    out = (len(input.group()) - 1) * ' ' + '\n'
    return out


def clean_create(file_name):
    """
    处理掉开头的创建表和删除表语句
    """
    logging.info('删除文件创建语句 %s' % os.path.join(os.getcwd(), file_name))
    f = open(file_name, 'rb+')
    buffer_map = mmap.mmap(f.fileno(), length=buffer_size, access = mmap.ACCESS_WRITE, offset=0)
    # 读取前4096字节
    text = buffer_map.read(buffer_size)
    # 替换内容是一个函数replace_function
    ret = re.sub(re_intance, replace_function, text)
    # 返回nmap的启示位置
    buffer_map.seek(0)
    # 替换结果写入nmap中
    buffer_map.write(ret)
    buffer_map.flush()
    f.close()

```
