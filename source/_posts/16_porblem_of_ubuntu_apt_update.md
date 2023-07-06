---
title: 解决ubuntu apt update时出错的问题
date: 2017-10-02 14:30:15
tags: [ubuntu,geek]
---

错误类似：
```
Reading package lists... Done
E: Problem executing scripts APT::Update::Post-Invoke-Success 'if /usr/bin/test -w /var/cache/app-info -a -e /usr/bin/appstreamcli; then appstreamcli refresh > /dev/null; fi'
E: Sub-process returned an error code
```

解决方法如下：

`sudo pkill -KILL appstreamcli`

`wget -P /tmp https://launchpad.net/ubuntu/+archive/primary/+files/appstream_0.9.4-1ubuntu1_amd64.deb https://launchpad.net/ubuntu/+archive/primary/+files/libappstream3_0.9.4-1ubuntu1_amd64.deb`

`sudo dpkg -i /tmp/appstream_0.9.4-1ubuntu1_amd64.deb /tmp/libappstream3_0.9.4-1ubuntu1_amd64.deb`

执行完上述命令之后再次运行sudo apt-get update就不会再出现上面的错误。

参考：[https://askubuntu.com/questions/774986/appstreamcli-hanging-with-100-cpu-usage-during-update](https://askubuntu.com/questions/774986/appstreamcli-hanging-with-100-cpu-usage-during-update)

