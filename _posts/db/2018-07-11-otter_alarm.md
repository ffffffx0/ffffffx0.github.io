---
layout: post
title: "otter支持钉钉短信等报警"
keywords: ["otter"]
description: "otter"
category: "MySQL"
tags: ["otter","binlog"]
---

目前只支持邮件报警，发送邮件的代码在DefaultAlarmService里边

```
public void doSend(AlarmMessage data) throws Exception {
    SimpleMailMessage mail = new SimpleMailMessage(); // 只发送纯文本
    mail.setFrom(username);
    mail.setSubject(TITLE);// 主题
    mail.setText(data.getMessage());// 邮件内容
    String receiveKeys[] = StringUtils.split(StringUtils.replace(data.getReceiveKey(), ";", ","), ",");

    SystemParameter systemParameter = systemParameterService.find();
    List<String> mailAddress = new ArrayList<String>();
    for (String receiveKey : receiveKeys) {
        String receiver = convertToReceiver(systemParameter, receiveKey);
        String strs[] = StringUtils.split(StringUtils.replace(receiver, ";", ","), ",");
        for (String str : strs) {
            if (isMail(str)) {
                if (str != null) {
                    mailAddress.add(str);
                }
            } else if (isSms(str)) {
                // do nothing
            }
        }
    }

    if (!mailAddress.isEmpty()) {
        mail.setTo(mailAddress.toArray(new String[mailAddress.size()]));
        doSendMail(mail);
    }
}
```

首先会从报警规则里的receiveKey，获取receiver，这个receiver应该是按";"分割的，这里先替换“;”为“,”，然后按照","分割

所以只要增加钉钉，手机号码的代码就好了

```
public void doSend(AlarmMessage data) throws Exception {
    SimpleMailMessage mail = new SimpleMailMessage(); // 只发送纯文本
    mail.setFrom(username);
    mail.setSubject(TITLE);// 主题
    mail.setText(data.getMessage());// 邮件内容
    String receiveKeys[] = StringUtils.split(StringUtils.replace(data.getReceiveKey(), ";", ","), ",");

    SystemParameter systemParameter = systemParameterService.find();
    List<String> mailAddress = new ArrayList<String>();
    for (String receiveKey : receiveKeys) {
        String receiver = convertToReceiver(systemParameter, receiveKey);
        String strs[] = StringUtils.split(StringUtils.replace(receiver, ";", ","), ",");
        for (String str : strs) {
            if (isMail(str)) {
                if (str != null) {
                    mailAddress.add(str);
                }
            } else if (isSms(str)) {
                // do nothing
            }
        }
    }

    if (!mailAddress.isEmpty()) {
        mail.setTo(mailAddress.toArray(new String[mailAddress.size()]));
        doSendMail(mail);
    }
    doSendDDmsg(String.format("from otter alarm msg: %s",data.getMessage()));
}
```

### 坑！坑！坑！

更新完之后出现

```
Caused by: javax.mail.MessagingException: Could not connect to SMTP host: smtp.exmail.qq.com, port: 25
	at com.sun.mail.smtp.SMTPTransport.openServer(SMTPTransport.java:1961) ~[mail-1.4.7.jar:1.4.7]
	at com.sun.mail.smtp.SMTPTransport.protocolConnect(SMTPTransport.java:654) ~[mail-1.4.7.jar:1.4.7]
	at javax.mail.Service.connect(Service.java:295) ~[mail-1.4.7.jar:1.4.7]
	at org.springframework.mail.javamail.JavaMailSenderImpl.doSend(JavaMailSenderImpl.java:389) ~[spring-context-support-3.1.2.RELEASE.jar:3.1.2.RELEASE]
	... 12 common frames omitted
Caused by: javax.net.ssl.SSLException: Unrecognized SSL message, plaintext connection?
	at sun.security.ssl.InputRecord.handleUnknownRecord(InputRecord.java:710) ~[na:1.8.0_111]
	at sun.security.ssl.InputRecord.read(InputRecord.java:527) ~[na:1.8.0_111]
	at sun.security.ssl.SSLSocketImpl.readRecord(SSLSocketImpl.java:973) ~[na:1.8.0_111]
	at sun.security.ssl.SSLSocketImpl.performInitialHandshake(SSLSocketImpl.java:1375) ~[na:1.8.0_111]
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1403) ~[na:1.8.0_111]
	at sun.security.ssl.SSLSocketImpl.startHandshake(SSLSocketImpl.java:1387) ~[na:1.8.0_111]
	at com.sun.mail.util.SocketFetcher.configureSSLSocket(SocketFetcher.java:549) ~[mail-1.4.7.jar:1.4.7]
	at com.sun.mail.util.SocketFetcher.createSocket(SocketFetcher.java:354) ~[mail-1.4.7.jar:1.4.7]
	at com.sun.mail.util.SocketFetcher.getSocket(SocketFetcher.java:211) ~[mail-1.4.7.jar:1.4.7]
	at com.sun.mail.smtp.SMTPTransport.openServer(SMTPTransport.java:1927) ~[mail-1.4.7.jar:1.4.7]
```

按理没啥改动，就增加了http的jar包

```
$ ls lib/httpc*
lib/httpclient-4.5.2.jar  lib/httpcore-4.4.4.jar
```

manager.biz这个包复原回去又能调用，最后看了下腾讯企业邮箱的ssl端口应该是465，换了端口好了

```
otter.manager.monitor.email.host = smtp.exmail.qq.com
otter.manager.monitor.email.stmp.port = 465
```

期间还试过不改端口25，就是不使用ssl,去掉socketFactory的配置也ok

```
<bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
    <property name="host" value="${otter.manager.monitor.email.host}" />
    <property name="username" value="${otter.manager.monitor.email.username}" />
    <property name="password" value="${otter.manager.monitor.email.password}" />
    <property name="defaultEncoding" value="UTF-8" />
    <property name="javaMailProperties">
        <props>
            <prop key="mail.smtp.auth">true</prop>
            <prop key="mail.smtp.timeout">25000</prop>
            <prop key="mail.smtp.port">${otter.manager.monitor.email.stmp.port:465}</prop>
            <!-- 
            <prop key="mail.smtp.socketFactory.port">${otter.manager.monitor.email.stmp.port:465}</prop>
            <prop key="mail.smtp.socketFactory.fallback">false</prop>
            <prop key="mail.smtp.socketFactory.class">javax.net.ssl.SSLSocketFactory</prop>
             -->
            <prop key="mail.smtp.starttls.enable">${otter.manager.monitor.mail.smtp.starttls.enable}</prop>

            <prop key="mail.smtp.ssl.enable">${otter.manager.monitor.mail.smtp.ssl.enable}</prop>
        </props>
    </property>
</bean>
```

虽然看起来是端口问题，但是不确定为何在没有引入httpclient后可以使用25端口正常发送邮件


#### 注意

钉钉自定义机器人的消息长度貌似有限制，官网说是content最大长度5000，实际测试貌似是20000

```
String ddMsg = msg.substring(0,20001);
```
