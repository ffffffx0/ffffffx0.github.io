---
title: 记一次Aviator使用不当导致的线上OOM
tagline: ""
category : Java
layout: post
tags : [Java, File, tools，OOM]
---

### 问题描述
1. 是的,没错,正如题中所说是Aviator使用不当造成的OOM,为何使用不当后边说
2. 项目上线几个月了,发现云平台时不时会重启server,一般出现在某些高峰期，比如早上7-9点,晚上12点
3. 通过-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath,配置可以获取到dump文件
4. 发现每次dump都是自带监控产生的oom，自带监控的实现是基于Prometheus


###  oom堆栈信息
具体监控如下
```
"pool-3-thread-2" prio=5 tid=124 RUNNABLE
    at java.lang.OutOfMemoryError.<init>(OutOfMemoryError.java:48)
    at sun.util.resources.TimeZoneNames.getContents(TimeZoneNames.java:47)
    at sun.util.resources.OpenListResourceBundle.loadLookup(OpenListResourceBundle.java:137)
    at sun.util.resources.OpenListResourceBundle.loadLookupTablesIfNecessary(OpenListResourceBundle.java:128)
    at sun.util.resources.OpenListResourceBundle.handleKeySet(OpenListResourceBundle.java:96)
    at java.util.ResourceBundle.containsKey(ResourceBundle.java:1824)
       Local Variable: sun.util.resources.TimeZoneNames#1
    at sun.util.locale.provider.LocaleResources.getTimeZoneNames(LocaleResources.java:263)
       Local Variable: sun.util.locale.provider.LocaleResources#1
       Local Variable: sun.util.resources.en.TimeZoneNames_en#1
       Local Variable: java.lang.String#140268
    at sun.util.locale.provider.TimeZoneNameProviderImpl.getDisplayNameArray(TimeZoneNameProviderImpl.java:124)
    at sun.util.locale.provider.TimeZoneNameProviderImpl.getDisplayName(TimeZoneNameProviderImpl.java:99)
    at sun.util.locale.provider.TimeZoneNameUtility$TimeZoneNameGetter.getName(TimeZoneNameUtility.java:240)
    at sun.util.locale.provider.TimeZoneNameUtility$TimeZoneNameGetter.getObject(TimeZoneNameUtility.java:198)
    at sun.util.locale.provider.TimeZoneNameUtility$TimeZoneNameGetter.getObject(TimeZoneNameUtility.java:184)
    at sun.util.locale.provider.LocaleServiceProviderPool.getLocalizedObjectImpl(LocaleServiceProviderPool.java:294)
       Local Variable: java.lang.String#39894
       Local Variable: java.util.HashSet#79
       Local Variable: sun.util.locale.provider.TimeZoneNameProviderImpl#1
       Local Variable: sun.util.locale.provider.TimeZoneNameUtility$TimeZoneNameGetter#1
    at sun.util.locale.provider.LocaleServiceProviderPool.getLocalizedObject(LocaleServiceProviderPool.java:265)
    at sun.util.locale.provider.TimeZoneNameUtility.retrieveDisplayNamesImpl(TimeZoneNameUtility.java:166)
       Local Variable: java.lang.String[]#4784
       Local Variable: java.util.Locale#16
       Local Variable: sun.util.locale.provider.LocaleServiceProviderPool#3
    at sun.util.locale.provider.TimeZoneNameUtility.retrieveDisplayName(TimeZoneNameUtility.java:137)
    at java.util.TimeZone.getDisplayName(TimeZone.java:400)
       Local Variable: java.lang.String#696
       Local Variable: sun.util.calendar.ZoneInfo#985
    at java.text.SimpleDateFormat.subFormat(SimpleDateFormat.java:1271)
    at java.text.SimpleDateFormat.format(SimpleDateFormat.java:966)
       Local Variable: java.lang.StringBuffer#8
       Local Variable: java.text.DontCareFieldPosition$1#1
       Local Variable: java.text.SimpleDateFormat#190
    at java.text.SimpleDateFormat.format(SimpleDateFormat.java:936)
    at java.text.DateFormat.format(DateFormat.java:345)
    at sun.net.httpserver.ExchangeImpl.sendResponseHeaders(ExchangeImpl.java:212)
       Local Variable: java.io.BufferedOutputStream#43
       Local Variable: sun.net.httpserver.PlaceholderOutputStream#1
       Local Variable: sun.net.httpserver.ExchangeImpl#1
    at sun.net.httpserver.HttpExchangeImpl.sendResponseHeaders(HttpExchangeImpl.java:86)
    at metrics.exporter.PrometheusMetricsExporter.lambda$startServer$0(PrometheusMetricsExporter.java:88)
```
这里翻了下Prometheus看起来来也没啥特殊的，跟中间件的对了下，好像也看不出啥问题，但是发现dump日志只有50M左右,与实际-Xmx2g相差剩余
没招只好找自己程序的代码的BUG,CR了几次都没有发现问题，没办法oom了只好看gc日志了，发现fullgc一直在进行，占用的时间也不少

#### 看下gc日志
![gc日志监控](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/11.png)

这里看出来其实是有fullgc的，还很频繁，因为dump依旧是那个50多M的文件，oom发生的位置也的都是Prometheus监控SimpleDateFormat.format时间，这个按里确实不太对

于是又去CR代码，这次有点收获，发现有个老项目迁移过来的代码原本是要用单例的，可能是当时迁移赶时间，没有用单例，但是用的static修饰了该方法，不合理先按代码review规范改一版上线吧，于是把static方法改成了DCL单例即时上线了，神奇的是上线完之后立马就好了，fullgc没了。没有使用单例可能导致对象增多理论上也不一定就会oom，但是dump文件没有体现

### 其他jvm监控现象
#### 类加载指标
![类加载](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/12.png)
#### 其他正常程序类加载指标
![其他正常程序类加载](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/13.png)
又仔细看了DCL内的代码逻辑，有用到aviator规则表达式,跟了下它的实现
```
AviatorEvaluator.compile(script);
public static Expression compile(String expression) {
    return compile(expression, false);
}
public static Expression compile(String expression, boolean cached) {
    return getInstance().compile(expression, cached);
}
public Expression compile(String expression, boolean cached) {
    return this.compile(expression, expression, cached);
}
public Expression compile(String cacheKey, String expression, boolean cached) {
    if (expression != null && expression.trim().length() != 0) {
        if (cacheKey != null && cacheKey.trim().length() != 0) {
            if (!cached) {
                return this.innerCompile(expression, cached);
            } else {
            //省略部分代码
            }
       }
    }
}
private Expression innerCompile(String expression, boolean cached) {
    ExpressionLexer lexer = new ExpressionLexer(this, expression);
    CodeGenerator codeGenerator = this.newCodeGenerator(cached);
    ExpressionParser parser = new ExpressionParser(this, lexer, codeGenerator);
    Expression exp = parser.parse();
    if (this.getOptionValue(Options.TRACE_EVAL).bool) {
        ((BaseExpression)exp).setExpression(expression);
    }

    return exp;
}
```
重点是这个newCodeGenerator
```
public CodeGenerator newCodeGenerator(boolean cached) {
    AviatorClassLoader classLoader = this.getAviatorClassLoader(cached);
    return this.newCodeGenerator(classLoader);
}
```
这里默认cached是false，也就是每次调用这个规则表达式的时候都会去重新classloader一遍
```
public AviatorClassLoader getAviatorClassLoader(boolean cached) {
    return cached ? this.aviatorClassLoader : new AviatorClassLoader(Thread.currentThread().getContextClassLoader());
}
```

看到这里大概能跟前面类加载指标对的上了，为什么类加载指标正常程序是15000左右，这个大概能到4,50000

这里差不多了

### 修复后效果
#### 看下FullGC明显没了
![gc日志监控](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/2.png)
#### 类加载指标
![类加载](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/3.png)

其实还忽略了个问题,那就是console.log,搜了下确实有oom的，不过是java.lang.OutOfMemoryError: Metaspace
看到没是metaspace, 这就对上了，看下修复前后对比把

![gc日志监控](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/1.png)








