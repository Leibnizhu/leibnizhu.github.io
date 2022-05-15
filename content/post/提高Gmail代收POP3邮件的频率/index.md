---
title: 提高Gmail代收POP3邮件的频率
date: 2016-08-12T15:02:43+08:00
tags:
- Gmail
- 效率
---

工作电脑是Linux系统，没有什么顺手的Email客户端（ThunderBird会反复接收同样的邮件、最后收件箱里一大堆重复的邮件真不想再吐槽了），而公司邮箱是疼讯企业邮箱，WebUI不忍直视，所以选择了Gmail的POP3代收，Chrome长期开着web版，打开消息通知。

问题就来了，Gmail会根据POP3代收邮箱的来件频率动态调整其收件频率，这也是可以理解的，因为POP3代收没法推送只能主动收件，如果可以随便设收件频率，那大家都设成1分钟一收，那Gmail服务器的负载也太大了。虽然这样的设定合理，而我的邮箱收件频率也不算高，最终大概是半小时收一次，有时候甚至是40-50min，这样突发的紧急邮件就很容易错过。

Google了一下，解决方案有三：
1. 订阅大量邮件，刺激Gmail提高收件频率；同时在Gmail里设置过滤器将这些订阅的邮件删除。但是要定期去被代收的邮箱里去清理，也要小心被代收的邮箱容量问题。
2. 与解决方案1类似，自己写脚本定时给被代收邮箱发邮件，但容易被被代收邮箱拉黑，其余缺点也一样有。
3. Chrome安装[Gmail POP3 Checker插件](http://www.danielslaughter.com/projects/gmail-pop3-checker-for-greasemonkey/#install)（需要先安装TemperMonkey）,可以设置POP3代收频率。

前两种方法显然太烂，选择第3种。安装插件后发现免费版不能自由设定收件频率，只能是默认的12分钟，虽然比Gmail自己的快了不少，但还是不满足。而捐献的话至少5刀，太贵了，1刀的话我就给了。于是想办法自己弄吧。

我的解决方案很简单，在Chrome控制台写入一段JS代码，定期进入POP3代收管理页面，触发手动收件的点击事件，最后再返回到收件箱页面即可，代码如下：
```javascript
window.setInterval(function(){
    window.location.href="https://mail.google.com/mail/u/0/#settings/accounts";
    var exitTime = new Date().getTime() + 5000; //5秒后再执行，以免页面加载慢
    while (new Date().getTime() < exitTime) {;}
    var checkEmails = document.getElementsByClassName('rP sA');
    for(var i =0; i < checkEmails.length; i++){
        checkEmails[i].click();
    }
    window.location.href="https://mail.google.com/mail/u/0/#inbox";
}, 3*60*1000);
```

其实可以写成Chrome扩展的，我就懒了，毕竟Chrome一直开着，很少要重新开Gmail页面。
