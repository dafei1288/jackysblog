---
layout: post
category: 技术
title: apache整合tomcat
tagline: by Jacky
duoshuo: true
tags: [java,tomcat,apache,中文乱码]
date: 2014-09-02
---

最近在使用apache整合tomcat,使用ajp连接.

但是在发布的时候,突然发现问题了.

然后排查问题,最后发现是某些必要文件找不到,

那么是中文路径?文件编码?

最后发现是在URL传递的时候出现了问题.

##URL传递乱码问题

通过端口访问TOMCAT的时候一切正常,

但是通过域名访问就失败了.

最后发现是JK+AJP传输URL时候出现了中文

解决方法经过baidu发现都不管用.

最后找到解决问题的方法:

1.tomcat端修改
```xml
<Connector port="8085" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443"
    URIEncoding="UTF8" />
                 
<Connector port="8009" protocol="AJP/1.3" redirectPort="8443"
  	URIEncoding="UTF8" />
```

2.apache端修改

>  JkOptions  -ForwardDirectories

##关于配置问题

感觉还是自己编译mod_jk靠铺点,

在http://tomcat.apache.org/download-connectors.cgi 这里可以下载到JK的最新原码

然后native下

> ./configure --with-apxs=/opt/lampp/bin/apxs 

> make

> make install

在apache2/mod_jk.so


    > last update @ 2014-09-03 9:03 am  by jack







