发展过程

1. 最早的 HTTP 由 Tim Berners-Lee 于 1989 年开始开发，后由万维网协会（World Wide Web Consortium，W3C）和互联网工程任务组（Internet Engineering Task Force, IETF）共同制定，
2. http 0.9：1991年推出。因为那个年代互联网还在普及，加上网速带宽低，所以 HTTP 0.9 只支持 GET 请求，只支持传输文本。
3. http 1.0：1996 年推出，增加 POST 和 HEAD 方法，支持传输超文本数据，报文格式发生了变化，




### 1.0 和 1.1 的区别

1、HTTP 1.1 需要指定一个叫 Host 头信息；而 HTTP 1.0 官方不需要这个 Host 头，但加了也无妨，很多如代理的程序都乐意在报文中看到一个 Host 头不管是哪个版本的 HTTP 协议。如：

```
GET / HTTP/1.1
Host: www.blahblahblahblah.com
```

2、HTTP 1.1 允许持久连接，意味着可以在一个连接中进行多次 request 得到多个 response，在 HTTP 1.0 中，每个连接对应一个 request/response 对，每次 response 完连接就会关闭，因为 TCP 慢启动，这会导致很多效率问题。

3、HTTP/1.1 中新加了一个 OPTIONS 方法，可用于实现跨域（Cross Origin Resource）

4、HTTP/1.0 通过 If-Modified-Since 头信息来支持缓存，HTTP 1.1 通过 entity tag 扩展了缓存支持，如果两个资源是一样的，那么他们会有一个一样的 e tag。HTTP 1.1 还加入了 If-Unmodified-Since, If-Match, If-None-Match conditional 头信息，与缓存相关的还有，加了 个  Cache-Control 头。

5、HTTP/1.1 中新加了一个 100 状态码，用于避免客户端当客户端不确定服务器是否可以或被授权去处理该请求时发送一个大的请求，这种情况下只发送请求头，然后服务器返回 100 Continue，然后客户端再继续发送请求体。

其他区别

- 额外的新状态码
- 连接头
- 高级压缩支持


相关文档

1. HTTP 1.0 官方规范：https://www.w3.org/Protocols/HTTP/1.0/spec.html
2. HTTP 1.1 官方规范：https://www.w3.org/Protocols/HTTP/1.1/spec.html
3. HTTP 2.0 官方规范：https://httpwg.org/specs/rfc7540.html
4. 头部压缩方式：https://datatracker.ietf.org/doc/html/rfc7541#page-33
5. https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol
6. https://datatracker.ietf.org/doc/html/rfc7231
7. HTTP 0.9：https://www.w3.org/Protocols/HTTP/AsImplemented.html
8. https://datatracker.ietf.org/doc/html/rfc1945
