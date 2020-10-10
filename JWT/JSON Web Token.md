JWT全称叫做JSON Web Token，是一种认证的解决方案。和Cookie-Session模式分别用于不同应用场景。



### Cookie-Session的认证方式

Cookie-Session的认证流程一般如下：

1、用户向服务器发送用户名和密码。

2、服务器验证通过后，在当前对话（session）里面保存相关数据，比如用户角色、登录时间等等。

3、服务器向用户返回一个 session_id，写入用户的 Cookie。

4、用户随后的每一次请求，都会通过 Cookie，将 session_id 传回服务器。

5、服务器收到 session_id，找到前期保存的数据，由此得知用户的身份。



Cookie-Session的认证方式的问题：

1、如果服务是集群，需要一个中间组件保存Session，比如Redis，或者数据库。可能成为性能瓶颈。

2、对于移动端，处理Cookie不太方便。



### JWT是什么？

JWT差不多长这样:

```
eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJodHRwOi8vZXhhbXBsZS5vcmciLCJhdWQiOiJodHRwOi8vZXhhbXBsZS5jb20iLCJpYXQiOjEzNTY5OTk1MjQsIm5iZiI6MTM1NzAwMDAwMCwiZXhwIjoxNDA3MDE5NjI5LCJqdGkiOiJpZDEyMzQ1NiIsInR5cCI6Imh0dHBzOi8vZXhhbXBsZS5jb20vcmVnaXN0ZXIiLCJjdXN0b20tcHJvcGVydHkiOiJmb28iLCJuYW1lIjoiUm9iIE1jTGFydHkiLCJpZCI6Nzh9.-3BnaA1XRiKh8e7zy9ZRTETf8VngoypqrVW98oQnH4w
```

由"."将字符串分隔为3部分：

* Header
* Payload
* Signature

Header是元数据JSON base64URL之后得到的字符串，原JSON一般这样：

```json

{
  "alg": "HS256",
  "typ": "JWT"
}
```

表示签名算法是HS256，token的类型是JWT。

Payload是想传递的数据，可以仅仅是一个user_id，也可以带有非敏感的业务信息（因为base64URL并不是加密，很容易得到原文）。

```json
{  
  "iss": "http://example.org",  
  "aud": "http://example.com",  
  "iat": 1356999524,  
  "nbf": 1357000000,  
  "exp": 1407019629,  
  "jti": "id123456",  
  "typ": "https://example.com/register",  
  "custom-property": "foo",  
  "name": "Rob McLarty",  
  "id": 78  
}
```

> 官方定义了7个Payload标准字段
>
> - iss (issuer)：签发人
> - exp (expiration time)：过期时间
> - sub (subject)：主题
> - aud (audience)：受众
> - nbf (Not Before)：生效时间
> - iat (Issued At)：签发时间
> - jti (JWT ID)：编号

Signature是对前两部分的签名：

```
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

secret是秘钥，只在服务端保存。这样，如果前两部分又被篡改，服务端验证的时候，签名就对不上了。签名保证了数据不会被篡改。



### JWT的优点和缺点

有了JWT这样一个结构，服务端就不需要保存session了。当客户端通过用户名密码认证后，服务端就返回给客户端一个JWT，客户端持有这个JWT，每次请求的时候带上。一般通过HTTP请求的Header：

```http
Authorization: Bearer <token>
```

服务集群可以共享一个秘钥，这样认证的任务都可以自己完成，不需要互相访问，提高性能。

JWT有过期时间字段，因此服务端不用对过期进行任何操作，验证token的时候发现过期就是过期了。然而，这也是缺点，服务端不能主动过期一个token。(如果硬想满足主动过期的需求，需要使用一个类似黑名单的管理中心，就又回到session的方案上去了)。不能主动注销token，对于需要退出登录的应用很致命，没有太完美方案，都会回到session方案上去。

对于移动端，不需要操作Cookie，非常友好。

> 网上有说JWT可以解决跨域认证问题，其实解决不了。A域名下拿到的token还得想办法共享给B域名，不是无感知的。



总之，JWT看起来带来最大的好处就是服务端简洁，能提高性能。代价是不能注销认证，只能等待自然过期，需要谨慎使用，因为性能问题可能没到那么严重，但是功能不能满足一般不太行。



### 参考文献

1. https://robmclarty.com/blog/what-is-a-json-web-token

2. https://www.ruanyifeng.com/blog/2018/07/json_web_token-tutorial.html

3. https://juejin.im/entry/6844903598187347976

