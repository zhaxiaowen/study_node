# 文件处理

#### 多进程copy文件

```
# -*- coding: utf-8 -*-
import multiprocessing
import os
import shutil

def jugment_file(file_dir):
    all_file = os.listdir(file_dir)
    for i in all_file:
        name = file_dir + '/' + i
        if os.path.isdir(name):
            print("dir:", name)
        else:
            print("file:", name)

def file_name(src, dest):
    dict1 = dict()
    for root, dirs, files in os.walk(src):
        for file in files:
            source_path = os.path.join(root, file)
            file_path = source_path.split(src)
            if len(file_path) != 2:
                continue
            if file.endswith(".html"):
                target_path = os.path.join(dest, "html/", file_path[1])
            elif file.endswith(".pdf"):
                target_path = os.path.join(dest, "pdf/", file_path[1])

            elif file.endswith(".mp3"):
                target_path = os.path.join(dest, "mp3/", file_path[1])

            elif file.endswith(".mp4"):
                target_path = os.path.join(dest, "mp4/", file_path[1])

            else:
                target_path = os.path.join(dest, "other/", file_path[1])

            dict1[source_path] = target_path
    return dict1

def dict_name(file_dir):
    for root, dirs, _ in os.walk(file_dir):
        if len(dirs) != 0:
            print(root, dirs)

def copy_file(src, dst):
    dist_path = os.path.dirname(dst)
    if not os.path.exists(dist_path):
        os.makedirs(dist_path)
    shutil.copy(src, dst)

def main():
    src_path = '/study/3. 视频 /'
    # dest_path = '/wd/4t_backup/'
    dest_path = '/study/test/'
    result = file_name(src_path, dest_path)
    # q = multiprocessing.Manager().Queue()
    po = multiprocessing.Pool(10)
    for key, value in result.items():
        po.apply_async(copy_file, args=(key, value))
    po.close()
    po.join()
    print(" 完成 ")

if __name__ == '__main__':
    main()
```



#### python

```
# -*- coding: utf-8 -*-

import os

def jugment_file(file_dir):
    all_file = os.listdir(file_dir)
    for i in all_file:
        name = file_dir + '/' + i
        if os.path.isdir(name):
            print("dir:", name)
        else:
            print("file:", name)

def file_name(file_dir):
    for root, dirs, files in os.walk(file_dir):
        for file in files:
            file_path = os.path.join(root, file)
            print(file_path)

def dict_name(file_dir):
    for root, dirs, _ in os.walk(file_dir):
        if len(dirs) != 0:
            print(root, dirs)

src_path = '/study/'
dest_path = '/wd/4t_backup/'
dict_name(src_path)
```

#### shell

```
#!/bin/bash

dir='/study/'
dest='/wd/4t_backup/'
num=20
depth='2 1' #归递目录深度
#depth='4 3 2 1' #归递目录深度

# 检测目录是否有变化
#/usr/local/bin/inotifywait -mrq -e modify,delete,create,attrib,move /study|while read files
#do
# echo "hello"
#done
for l in $depth; do
  todo=$(find -L $dir -maxdepth $l -mindepth $l -type d)
  IFS_old=$IFS
  IFS=$'\n'
  for i in $todo; do
    now_num=$(ps axw | grep rsync | grep -v 'grep' | wc -l)
    #      echo "${i/$dir/}::$now_num"
    while [ $now_num -ge $num ]; do
      echo "wait 10s"
      sleep 10
      now_num=$(ps axw | grep rsync | grep -v 'grep' | wc -l)
    done
    if [ -f "$i" ] || [ -d "$i" ]; then
      echo "$i : /wd/other_backup/$i"
    else
      echo " 不存在 :$i"
    fi

# /usr/bin/rsync -avzh --delete --progress $i/ /wd/other_backup/$i &
  done
  IFS=$IFS_old
done

# 最后再校验一遍
#while true; do
#	sleep 5
#	now_num=`ps axw | grep rsync  | grep -v 'grep' | wc -l`
#	if [ $now_num -lt 1 ]; then
#		echo " 同步完成 "
#		break
# 	fi
#done

```

