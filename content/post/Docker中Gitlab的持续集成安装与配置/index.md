---
title: Docker中Gitlab的持续集成安装与配置
date: 2017-04-26T15:02:26+08:00
tags:
- Gitlab
- 持续集成
- Tomcat
image: docker-gitlab.png
---

# 安装Gitlab-CI-Runner
## 下载
根据[官方的说法](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner)，Gitlab-CI-Runner9.0以后的版本需要GitLab9.0以上版本支持，我们目前部署的GitLab是8.x，所以需要下载旧版。  
最新版：  
```bash
sudo wget -O /usr/local/bin/gitlab-runner https://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-ci-multi-runner-linux-amd64
```
旧版（如v1.11.0）：  
```bash
sudo wget -O /usr/local/bin/gitlab-runner https://gitlab-ci-multi-runner-downloads.s3.amazonaws.com/v1.11.0/binaries/gitlab-ci-multi-runner-linux-amd64
```

## 安装配置
```bash
sudo chmod +x /usr/local/bin/gitlab-runner
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
sudo gitlab-runner register
```
执行上面最后一句命令的时候，会要求输入网址和秘钥，此时打开Gitlab页面，进项目，右上角配置按钮-Runners，再按此时页面上给出的来填。  
还会要求输入名称和标签之类的信息，到最后会提示输入运行环境之类，我们Gitlab是在docker上，但Runner和Gitlab在同一个docker容器中的，就是在Gitlab调用的角度上来看，Runner并不是在docker中，所以运行环境那里选shell就行。  

## 运行
```bash
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start
```

# 配置项目的CI脚本
在项目根目录创建文件.gitlab-ci.yml，写入CI执行的脚本，具体参考[官方文档](http://docs.gitlab.com/ce/ci/yaml/README.html)  
此处以最简单的maven打包部署tomcat为例：  
```yaml
stages:
  - deploy
deploy:
  stage: deploy
  only:
    - dev
  script:
    - mvn clean
    - mvn package
    - cp target/FissionSales.war /var/opt/gitlab/webapps/
```
此处限定了dev分支的提交才会触发CI任务，打包后复制到指定文件夹里。  
此外，我们将打包成功的war包复制到`/var/opt/gitlab/webapps/`中，这样做是因为，Gitlab在docker中，而Tomcat在宿主机里，因为权限方面的问题，我只好让Gitlab的CI任务将war包放在docker的volume（已经配置了`/var/opt/gitlab/`的volume）中，然后在宿主机中通过定时任务，检查war包的版本，检查到新版本时复制到宿主机的Tomcat中进行部署，具体的检查脚本如下：
```bash
#!/bin/bash
a=`sudo stat -c %Y /var/lib/docker/volumes/gitlab-data/_data/webapps/***.war`
b=`date +%s`
b=$[b-a]
echo $b
if [ $b -le 60 ];
then
   sudo cp  /var/lib/docker/volumes/gitlab-data/_data/webapps/***.war /enviroment/apache-tomcat-8.0.33/webapps/
else
   echo "No new war package..."
fi
```
这个脚本通过crontab定时每分钟执行，所以检查war包的修改时间与当前时间相差小于60秒就会复制war包到tomcat的webapps中。  

# 配置Pipeline邮件通知
Gitlab默认CI Pipeline任务成功失败都会发邮件通知，这样或许会困扰到大家，所以修改Gitlab的源码，只让部署不成功的时候才发邮件通知。  
进入docker，编辑`/opt/gitlab/embedded/service/gitlab-rails/app/services/notification_service.rb`，在`pipeline_finished`方法的开头添加`return if pipeline.status == "success" `，如：
```ruby
def pipeline_finished(pipeline, recipients = nil)
    return if pipeline.status == "success"

    email_template = "pipeline_#{pipeline.status}_email"
    …………………………
```

