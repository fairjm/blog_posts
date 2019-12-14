---
title: "jetty8 text/plain默认字符编码的坑"
date: 2018/03/28 21:48:00
---
今天在测试一个content-type为`text/plain`的API时发现后端requestBody乱码了，而线上正常。  
自己本地使用jetty8版本，插件自带版本，而线上使用jetty9。  
最开始没有特别注意版本的差异，毕竟这个插件也用了很久了一直没问题，就先从请求分析起。    
检查了下发送的请求中没有设置charset，但项目里是设置了spring的编码过滤器，CharacterEncodingFilter中的逻辑：
```java
String encoding = getEncoding();
if (encoding != null) {
    if (isForceRequestEncoding() || request.getCharacterEncoding() == null) {
        request.setCharacterEncoding(encoding);
    }
    if (isForceResponseEncoding()) {
        response.setCharacterEncoding(encoding);
    }
}
```
encoding配置里设置为`UTF-8`，强制设置没有开启，所以在request没有设置字符编码时会进行设置。    
但奇怪的是乱码的请求里是先被设置成了`ISO-8859-1`，在想会不会是gateway转发影响，进行抓包分析但还是没有看见charset。   
远程debug了正常的测试环境发现字符编码没有被预先设置。    
这就非常尴尬了，虽然测试环境没有问题，但本地这个问题的确是存在的，万一上了预发环境也有问题就麻烦了，看了一会儿diff没发现有配置上的变更。  
在同事的提醒下，也只有jetty版本的原因了。   

拉取了下jetty8的源码（jetty-8.8.0-SNAPSHOT），查看了一下，发现的确有个坑。

`AbstractHttpConnection`中的`parsedHeader`负责解析header，其中对应charset的解析代码:
```java
case HttpHeaders.CONTENT_TYPE_ORDINAL:
    value = MimeTypes.CACHE.lookup(value);
    _charset=MimeTypes.getCharsetFromContentType(value);
    break;
```

解析出来的_charset会在`headerComplete`中设置进request内:
```java
if(_charset!=null)
    _request.setCharacterEncodingUnchecked(_charset);
```

主要的判断代码在`MimeTypes`中
```java
    public static String getCharsetFromContentType(Buffer value)
    {
        if (value instanceof CachedBuffer)
        {
            switch(((CachedBuffer)value).getOrdinal())
            {
                case TEXT_HTML_8859_1_ORDINAL:
                case TEXT_PLAIN_8859_1_ORDINAL:
                case TEXT_XML_8859_1_ORDINAL:
                    return StringUtil.__ISO_8859_1;

                case TEXT_HTML_UTF_8_ORDINAL:
                case TEXT_PLAIN_UTF_8_ORDINAL:
                case TEXT_XML_UTF_8_ORDINAL:
                case TEXT_JSON_UTF_8_ORDINAL:
                    return StringUtil.__UTF8;
            }
        }
        //下面是用来解析charset的
        int i=value.getIndex();
        final int end=value.putIndex();
        int state=0;
        int start=0;
        boolean quote=false;
        for (;i<end;i++)
        {
            final byte b = value.peek(i);

            if (quote && state!=10)
            {
                if ('"'==b)
                    quote=false;
                continue;
            }

            switch(state)
            {
                case 0:
                    if ('"'==b)
                    {
                        quote=true;
                        break;
                    }
                    if (';'==b)
                        state=1;
                    break;

                case 1: if ('c'==b) state=2; else if (' '!=b) state=0; break;
                case 2: if ('h'==b) state=3; else state=0;break;
                case 3: if ('a'==b) state=4; else state=0;break;
                case 4: if ('r'==b) state=5; else state=0;break;
                case 5: if ('s'==b) state=6; else state=0;break;
                case 6: if ('e'==b) state=7; else state=0;break;
                case 7: if ('t'==b) state=8; else state=0;break;

                case 8: if ('='==b) state=9; else if (' '!=b) state=0; break;

                case 9:
                    if (' '==b)
                        break;
                    if ('"'==b)
                    {
                        quote=true;
                        start=i+1;
                        state=10;
                        break;
                    }
                    start=i;
                    state=10;
                    break;

                case 10:
                    if (!quote && (';'==b || ' '==b )||
                        (quote && '"'==b ))
                        return CACHE.lookup(value.peek(start,i-start)).toString(StringUtil.__UTF8);
            }
        }
        if (state==10)
            return CACHE.lookup(value.peek(start,i-start)).toString(StringUtil.__UTF8);
        //默认路径
        return (String)__encodings.get(value);
    }
```
 简单来看首先检查是否是预设的一些content-type，如果是的话直接返回。  
 而传入的`text/plain`不在默认的范围内，接下里的代码是寻找charset或者"charset"，而传入的内容里并没有。  
 最后走到了`return (String)__encodings.get(value);`，通过上面的代码可以找到这个属性的设置:
```java
final ResourceBundle encoding = ResourceBundle.getBundle("org/eclipse/jetty/http/encoding");
final Enumeration i = encoding.getKeys();
while(i.hasMoreElements())
{
    final Buffer type = normalizeMimeType((String)i.nextElement());
    __encodings.put(type,encoding.getString(type.toString()));
}
```
使用了ResourceBundle，对应的资源文件内容：
```
text/html	= ISO-8859-1
text/plain	= ISO-8859-1
text/xml	= UTF-8
text/json   = UTF-8
```
... ...默认值为ISO-8859-1。
而jetty9呢，切回master分支，jetty9设置的代码直接在`Request`类中了，如下：
```java
    @Override
    public String getCharacterEncoding()
    {
        if (_characterEncoding==null)
            getContentType();
        return _characterEncoding;
    }



    @Override
    public String getContentType()
    {
        MetaData.Request metadata = _metaData;
        String content_type = metadata==null?null:metadata.getFields().get(HttpHeader.CONTENT_TYPE);
        if (_characterEncoding==null && content_type!=null)
        {
            MimeTypes.Type mime = MimeTypes.CACHE.get(content_type);
            String charset = (mime == null || mime.getCharset() == null) ? MimeTypes.getCharsetFromContentType(content_type) : mime.getCharset().toString();
            if (charset != null)
                _characterEncoding=charset;
        }
        return content_type;
    }
```
   类似的逻辑还是继续走`MimeTypes`的方法，但最后发生了变化:
```java
           if (state==10)
            return StringUtil.normalizeCharset(value,start,i-start);

        return null;
    }
```
已经没有那个默认逻辑，直接返回null了。  
这个改动已经非常久远了。    
避免踩这种坑，勤升版本很重要啊... ...  
感觉又荒废了时光...
