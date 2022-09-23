---
title: "同步Hugo至OSS"
date: 2022-08-16T21:17:11+08:00
hidden: false
draft: false
tags: ["Python", "Hugo"]
categories: ["计算机"]
keywords: ["hugo", "oss"]
description: ""
slug: "hugo-oss"
---

由于博客系统更换到了Hugo，简单的，静态数据也就直接放到了阿里云OSS，寻找同步方式，看到了不错的方案：

[Hugo发布博客自动同步到阿里云OSS云存储 · OdinXu的博客](https://odinxu.com/post/hugo-use-aliyun-oss/)

我对其主代码进行了改动，使之支持：删除OSS中多余的文件，优化本地md5检验方式。

另外，在使用命令 “hugo” 生成静态文件前，最好删除已经存在于目录 “public” 中的文件，这同样可以使用自动化程序，如下面简短的Python代码，写好文件后放在脚本执行就好了。

```Python
import shutil
shutil.rmtree("D:/hugo/my_site/public")
print("'public'已删除")
```

不单是Hugo数据，你可以用它来同步任何文件。
关于同步，官方有更详细的信息：

[阿里云OSS & SDK](https://help.aliyun.com/document_detail/52834.html)

以下是改动后的代码：

``` Python
# -- coding: utf-8 --**
import os
import time
import json
import hashlib
import oss2
from concurrent.futures import ThreadPoolExecutor, as_completed

OSS_CONFIG_accessKeyId = ""  # accessKeyId
OSS_CONFIG_accessKeySecret = ""  # accessKeySecret
OSS_CONFIG_endpoint = ""  # endpoint
OSS_CONFIG_bucketName = ""
OSS_CONFIG_localDir = "D:/example_dir/"  # 注意 localDir 必须 / 结尾

MAX_THREAD_COUNT = 10
MD5_CACHE_FILE = "local_file_md5.cache"
md5dict = {}
new_md5dict = {}


# 计算文件的md5值
def file_md5(filename):
    myhash = hashlib.md5()
    f = open(filename, 'rb')
    while True:
        b = f.read(8096)
        if not b:
            break
        myhash.update(b)
    f.close()
    return myhash.hexdigest()


# 使用SDK API接口，检查OSS里是否存在文件，
# 然后检查object_meta的 ETag 值
# 跟本地文件计算的 file_md5 值是否相同
# 来决定，要不要上传文件
def sync_local_file_to_aliyun_oss(local_file_name):
    oss_object_key = get_oss_object_key(local_file_name)  # 得到(与.cache值不同的)本地文件在oss格式下的name
    cur_file_md5 = new_md5dict[local_file_name]  # 本地文件的md5

    up = False
    if bucket.object_exists(oss_object_key):
        filemeta = bucket.get_object_meta(oss_object_key)
        if filemeta.headers['ETag'].lower().strip('"') != cur_file_md5:
            up = True
    else:
        up = True

    if up:
        print('uploading: ' + local_file_name)
        result = bucket.put_object_from_file(oss_object_key, local_file_name)
        if result.status != 200:
            new_md5dict[local_file_name] = 'upload error'
            return 'upload error, response information: ' + str(result)
        else:
            return "Up new ok : " + oss_object_key
    else:
        return "...fit... : " + oss_object_key  # 不需要上传文件，因为md5相同


# 本地哪一些文件的md5值，跟本地cache文件保存的md5值不相同
# 决定和OSS对比文件前，先自己本地md5对比一下，这样速度快。
def find_diff_md5_local_file(files):
    global md5dict
    if os.access(MD5_CACHE_FILE, os.F_OK):
        try:
            f = open(MD5_CACHE_FILE, 'r')
            md5dict = json.load(f)
        except Exception as e:
            md5dict = {}
            print(e)
    result = []
    for file in files:
        now_md5 = file_md5(file)
        new_md5dict[file] = now_md5
        md5c = md5dict.get(file)  # 文件的原MD5值
        if (md5c is None) or (now_md5 != md5c):
            result.append(file)
    return result


# 获取bucket的信息，官方有更详细的介绍
def get_oss_information():
    result = bucket.get_bucket_stat()
    # 获取Bucket的总存储量，单位为字节。
    print(f'OSS总存储量为：{result.storage_size_in_bytes/1024/1024}MB')
    # 获取Bucket中总的Object数量。
    print(f'OSS文件数量为：{result.object_count}')


# 判断是否有多余的文件于OSS
def whether_delete(local_file_names):
    oss_object_keys = local_file_name_to_oss(local_file_names)
    need_delete_files = []
    for file in oss2.ObjectIterator(bucket):
        if file.key not in oss_object_keys:
            need_delete_files.append(file.key)
    code = False
    if len(need_delete_files) != 0:
        code = True

    return code, need_delete_files


# 使用多线程删除文件
def delete_files(need_delete_files):
    fCurrent = 0
    fCount = len(need_delete_files)
    executor = ThreadPoolExecutor(max_workers=3)
    all_task = [executor.submit(delete_file, file) for file in need_delete_files]

    for future in as_completed(all_task):
        fCurrent += 1
        data = future.result()
        print("{}/{}\t: {}".format(fCurrent, fCount, data))


def delete_file(file):
    result = bucket.delete_object(file)
    if result.status != 200:
        new_md5dict[f'{OSS_CONFIG_localDir}{file}'] = 'delete error'
        return 'delete error, response information: ' + str(result)
    else:
        return "delete  ok : " + file


def get_local_file_names():
    files = []
    for dirpath, dirnames, filenames in os.walk(OSS_CONFIG_localDir):
        for filename in filenames:
            local_filename = os.path.join(dirpath, filename)
            if is_windows:
                local_filename = local_filename.replace('\\', '/')
            files.append(local_filename)
    return files

# 将多个文件的name格式从本地转换为OSS
def local_file_name_to_oss(local_file_names):
    oss_object_keys = []
    for local_file_name in local_file_names:
        oss_object_key = get_oss_object_key(local_file_name)
        oss_object_keys.append(oss_object_key)
    return oss_object_keys

# 单个name转换
def get_oss_object_key(local_filename):
    if local_filename.endswith('.DS_Store') or not os.path.isfile(local_filename):
        return "...skip...: " + local_filename
    oss_object_key = local_filename[len(OSS_CONFIG_localDir):]
    return oss_object_key


if __name__ == '__main__':
    time_start = time.time()
    is_windows = (os.name == 'nt')

    files = get_local_file_names()
    print(f"本地文件数量为：{len(files)}")
    time_end1 = time.time()

    auth = oss2.Auth(OSS_CONFIG_accessKeyId, OSS_CONFIG_accessKeySecret)
    bucket = oss2.Bucket(auth, OSS_CONFIG_endpoint, OSS_CONFIG_bucketName)

    time_end2 = time.time()
    print('登录OSS：{:.20f}秒'.format(time_end2 - time_end1))
    get_oss_information()

    code, need_delete_files = whether_delete(files)
    if code:
        print(f'需要OSS删除的文件数量为：{len(need_delete_files)}')
        delete_files(need_delete_files)
        time_end3 = time.time()
        print('删除耗时{:.4f}秒'.format(time_end3 - time_end2))
    else:
        print(f"需要OSS删除的文件数量为：0")

    files = find_diff_md5_local_file(files)
    fCount = len(files)
    print("需要OSS上传的文件数量为：{}".format(fCount))

    if fCount == 0:
        print("程序结束(删除.cache文件可强制进行OSS文件上传)")
        exit(0)

    time_end4 = time.time()

    fCurrent = 0
    print("启动{}个线程进行上传".format(MAX_THREAD_COUNT))
    executor = ThreadPoolExecutor(max_workers=MAX_THREAD_COUNT)
    all_task = [executor.submit(sync_local_file_to_aliyun_oss, file) for file in files]

    # 保存本地文件最新的MD5数据，无旧数据干扰
    if True:
        try:
            f = open(MD5_CACHE_FILE, 'w')
            json.dump(new_md5dict, f)
            print("New MD5dict is OK")
        except Exception as e:
            print(f"Save md5 cache faild:{e}")

    for future in as_completed(all_task):
        fCurrent += 1
        data = future.result()
        print("{}/{}\t: {}".format(fCurrent, fCount, data))

    time_end5 = time.time()
    print('上传完全部文件：{:.4f}秒'.format(time_end5 - time_end4))

    time_end_all = time.time()
    print('程序全部时间：{:.4f}秒'.format(time_end_all - time_start))
```





