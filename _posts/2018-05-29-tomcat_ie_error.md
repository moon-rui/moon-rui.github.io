---
layout: post
title: 异常解决：Invalid character found in the request target. The valid characters are defined in RFC 7230 and RFC 3986
key: 2018-05-29-tomcat_ie_error
tags: 
   - Tomcat
   - IE
---

在用IE浏览器访问接口链接时，页面显示400错误，同时发现后台日志报错：

```
java.lang.IllegalArgumentException: Invalid character found in the request target. The valid characters are defined in RFC 7230 and RFC 3986
```

这是因为Tomcat在 7.0.73, 8.0.39, 8.5.7 版本后，添加了对于http头的验证。

具体来说，就是添加了些规则去限制HTTP头的规范性。

`org.apache.tomcat.util.http.parser.HttpParser#IS_NOT_REQUEST_TARGET[]`中定义了一堆not request target

```
if(IS_CONTROL[i] || i > 127 || i == 32 || i == 34 || i == 35 || i == 60 || i == 62 || i == 92 || i == 94 || i == 96 || i == 123 || i == 124 || i == 125) {
                IS_NOT_REQUEST_TARGET[i] = true;
            }
```

转换过来就是以下字符(对应10进制ASCII看):

- 键盘上那些控制键:(<32或者=127)

- 非英文字符(>127)

- 空格(32)

- 双引号(34)

- `#`(35)

- `<`(60)

- `>`(62)

- `\`(92)

- `^`(94)

- `(96)

- `{`(123)

- `}`(124)

- `|`(125)

**解决方法1：**

配置tomcat的catalina.properties

添加或者修改：

tomcat.util.http.parser.HttpParser.requestTargetAllow=\|{}

上述配置允许URL中包含\|{}字符，如果包含中文就不能使用这种配置。

**解决方法2：**

更换tomcat版本，使用较低的版本。

**解决方法3：**

对请求链接进行编码转义，可以使用JS中的encodeURI和decodeURI函数。Chrome、Firefox等浏览器会对URL自动进行转义，而IE却不行。这也是只有在用IE访问的情况下会报错的原因。

**解决方法4：**

选择另外的参数传递方法，比如post方式。