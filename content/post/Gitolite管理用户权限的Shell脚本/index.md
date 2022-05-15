---
title: Gitolite管理用户权限的Shell脚本
date: 2016-07-20T20:32:32+08:00
tags:
- git
- gitolite
- Shell脚本
- 权限管理
---
最近给公司的git服务器从http协议升级为ssh协议的，同时增加了gitolite作为权限管理。但gitolite在增删用户公钥、增删用户对项目的权限、以及新增repository时的操作说不上繁杂但也是蛮机械的；又不想用gitlab之类重型的解决方案，所以选择了编写Shell脚本来实现。

## 增加用户公钥的脚本
```bash
#!/bin/bash
echo "增加/修改 $1 的公钥 $1.pub……"
cd /home/git/gitolite-admin
echo $2 > keydir/$1.pub

git add .
git commit -m "增加/修改 $1 的公钥 $1.pub……"
git push
```

## 删除用户公钥的脚本
```bash
#!/bin/bash
echo "删除 $1 的公钥 $1.pub……"
cd /home/git/gitolite-admin
rm keydir/$1.pub

git add .
git commit -m "删除 $1 的公钥 $1.pub……"
git push
```

## 增加用户对指定代码仓库的读写权限
```bash
#!/bin/bash
#增加指定用户对指定代码仓库的读写权限
#第1个参数为用户名，对应公钥pub文件的文件名
#第2个参数为repository名，对应/home/git/repositories下的文件夹名（不包含.git后缀）
echo "增加 $1 对项目 $2 的权限……"
cd /home/git/gitolite-admin

isTarget=0
row=0
#找到要修改的行
cat conf/gitolite.conf | while read line
do
    row=`expr $row + 1`
    if [[ "$isTarget" == 1 ]];
    then
        echo "$line"
        if [[ "$line" =~ "$1"  ]];
        then
            echo "$1 已经拥有 $2 项目的读写权限"
            exit 0
        else
            #正式修改
            reg="${row}s/$/&$1 /g"
            sed -i "${reg}" conf/gitolite.conf
            break
        fi
        isTarget=0
    fi

    if [[ $line =~ "$2" ]];
    then
        echo "${line}"
        isTarget=1
    fi
done

git add .
git commit -m "增加 $1 对项目 $2 的权限……"
git push
```

## 删除用户对指定代码仓库的读写权限
```bash
#!/bin/bash
#删除指定用户对指定代码仓库的读写权限
#第1个参数为用户名，对应公钥pub文件的文件名
#第2个参数为repository名，对应/home/git/repositories下的文件夹名（不包含.git后缀）
echo "删除 $1 对项目 $2 的权限……"
cd /home/git/gitolite-admin

isTarget=0
row=0
#找到要修改的行
cat conf/gitolite1.conf | while read line
do
    row=`expr $row + 1`
    if [[ "$isTarget" == 1 ]];
    then
        echo "$line"
        if [[ "$line" =~ "$1"  ]];
        then
            #删对应用户
            reg="${row}s/$1 //g"
            sed -i "${reg}" conf/gitolite1.conf
            break
        else
            echo "$1 原来就没有 $2 项目的读写权限"
            exit 0
        fi
        isTarget=0
    fi

    if [[ $line =~ "$2" ]];
    then
        echo "${line}"
        isTarget=1
    fi
done

git add .
git commit -m "删除 $1 对项目 $2 的权限……"
git push
```

## 新增一个代码仓库，并初始化其权限
```bash
#!/bin/bash
#新增一个代码仓库并初始化权限管理文件
#第1个参数为该项目初始分配的用户名，对应公钥pub文件的文件名
#第2个参数为repository名，对应/home/git/repositories下的文件夹名（不包含.git后缀）
#第3个参数为项目在权限配置文件中的注释
echo "新增项目 $2 并初始化……"
cd /home/git/repositories
if [ -d "$2.git" ];
then
    echo "项目 $2 已存在，请选择另一个项目名……"
    exit 0
fi
echo "no exists"
git init --bare "$2.git"

#修改权限文件，增加默认管理员权限和初始化的用户权限
echo  "## $3" >> /home/git/gitolite-admin/conf/gitolite1.conf
echo  "repo	$2" >> /home/git/gitolite-admin/conf/gitolite1.conf
echo  "	RW = $1 " >> /home/git/gitolite-admin/conf/gitolite1.conf
echo  "	RW+ = @admin" >> /home/git/gitolite-admin/conf/gitolite1.conf

git add .
git commit -m "增加项目 $2 并初始化，同时增加用户 $1 对其的读写权限……"
git push

#初始化.gitignore文件
cd /home/git/gittmp
git clone git@172.16.99.235:$2.git
cd $2
echo "#filter class/binary file" >> .gitignore
echo "/build/" >> .gitignore
echo "/target/" >> .gitignore
echo "/bin/" >> .gitignore
echo "" >> .gitignore
echo "## filter eclipse file" >> .gitignore
echo "*.classpath" >> .gitignore
echo "*.project" >> .gitignore
echo "/.settings/" >> .gitignore
echo "" >> .gitignore
echo "## filter IntelliJ IDEA file" >> .gitignore
echo "*.iml" >> .gitignore
echo "/.idea/" >> .gitignore
echo "" >> .gitignore
echo "#filter temp file" >> .gitignore
echo "*.tmp" >> .gitignore
echo "/~$*" >> .gitignore
echo ".~*" >> .gitignore
echo "*.log" >> .gitignore
git add .
git commit -m "初始化项目的.gitignore文件，忽略常见无用文件"
git push
echo "项目 $2 创建成功"
```

## 重命名项目并修改对应权限配置文件
```bash
#!/bin/bash
#修改代码仓库的名字，同时更新权限配置文件
#第1个参数为原repository名，对应/home/git/repositories下的文件夹名（不包含.git后缀）
#第2个参数为repository名，对应/home/git/repositories下的文件夹名（不包含.git后缀）
echo "重命名项目 $1 为 $2……"
cd /home/git/repositories
if [ ! -d "$1.git" ];
then   
    echo "项目 $1 不存在，请选择另一个项目名……"
		exit 0
fi
#更改目录名
mv "$1.git" "$2.git"

#修改权限文件，增加默认管理员权限和初始化的用户权限
sed -i "s/$1$/$2/g" /home/git/gitolite-admin/conf/gitolite.conf
echo "新的项目URL为：git@172.30.16.235:$2.git"

cd /home/git/gitolite-admin
git add .   
git commit -m "重命名项目 $1 为 $2……"   
git push
```
