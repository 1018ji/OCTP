# SSL/TLS attacks: Part 3 – BREACH Attack (BREACH 攻击)

## 简介

Browser Reconnaissance and Ex-filtration via Adaptive Compression of Hypertext (BREACH) Attack:
Previously we learnt how CRIME attacks SSL/TLS using SSL/TLS compression. Now we look at a more recent attack called the BREACH attack.

基于超文本自适应压缩算法的浏览器侦测与渗透，简称 [BREACH](https://en.wikipedia.org/wiki/BREACH)：之前我们了解了[CRIME](https://en.wikipedia.org/wiki/CRIME) 攻击使用 SSL/TLS 压缩攻击 SSL/TLS。现在我们看一种最近出现升级的攻击方法，叫做 BREACH 攻击。


BREACH attack is quite similar to CRIME attack with subtle differences. This attack also leverages compression to extract data from a SSL/TLS channel. However, its focus is not on SSL/TLS compression; rather it exploits HTTP compression. Here, the attack tries to exploit the compressed and encrypted HTTP responses instead of requests as it was the case with CRIME attack.
Note: HTTP response compression applies only to the body of the response and not the header.

BREACH 攻击与 CRIME 攻击非常类似，只有细微差别。这种攻击方法也是借用压缩从 SSL/TLS 隧道解析数据。但是他的重点不是 SSL/TLS 压缩，而是利用 HTTP 压缩。这里的攻击尝试使用压缩以及加密的 HTTP 响应替代 CRIME 攻击使用的 HTTP 请求方式。

注意：HTTP 响应压缩只适用于响应主体，而不适用于响应头。


BREACH attack is more powerful than CRIME since HTTP compression can’t be readily turned off. It targets HTTP responses by targeting the size of the HTTP response and extracting data from the HTTP response.

BREACH 攻击比 CRIME 攻击危害更大，因为 HTTP 压缩不能被轻易关闭。它确定 HTTP 响应通过 HTTP 响应的大小以及 HTTP 响应数据进行。


Prerequisites of the BREACH attack:
The application must support HTTP compression.
User input should be reflected in the response.
The attacker must be able to do a Man-in-the-middle (MITM) attack on the victim.
The HTTP response should have some secret information like CSRF token.
The attack works by injecting data into the HTTP request and analyzing the length of the HTTP responses. Any variation in length of the response indicates a successful guess.

BREACH 攻击的先决条件：
- 必须支持 HTTP 压缩
- 用户输入信息必须返回并显现在响应信息中
- 攻击者必须能对受害者进行中间人（[MITM](https://en.wikipedia.org/wiki/Man-in-the-middle_attack)）攻击
- HTTP 响应必须含有一些加密信息，例如 CSRF 令牌等

攻击工作原理是讲数据注入到 HTTP 请求并分析 HTTP 响应的长度，任何响应长度的变化都表示进行了成功的猜测。

## How BREACH attack works? (BREACH 攻击如何工作？)

Consider the following request to an SSL enabled web server

考虑以下对开启 SSL 的 Web 服务器的请求

```
GET /stuff/form.php?id=786345
```
This generates the following response

以上请求产生如下响应

```
<a href="form2.php?token=csvfdfcrvet343v">Go to form2</a>
<form target="https://example.com:443/stuff/everything.php?id=786345">
…………
```

Here we see the string id=786345 is reflected back in the response body. The attacker leverages this very functionality to guess the ‘token’ value character by character.

这里我们看到字符串 `id=786345` 返回并显现在响应中。攻击者将会利用这个功能一个一个字符的猜测 `token ` 的值

The attacker injects the token value in the id parameter. If the value of the attacker injected token matches that of the actual token then the length of the response decreases by the length of the duplicate strings due to HTTP compression. In this case the value will be 21 viz. the length of the string token=csvfdfcrvet343v.Initially the attacker only knows that the ‘token=’ part of the string. This the attacker can know by simply logging into the application himself.

攻击者注入 `token` 值与 `id` 参数中。如果攻击者注入 `token` 值与实际 `token` 值匹配，那么响应的长度会因 HTTP 压缩而减少重复字符串的长度。 在上文构造的情况下减少的长度为 21，即字符串 `token=csvfdfcrvet343v` 的长度。最开始攻击者只需要知道字符串 `token=`，这部分攻击者可以通过登录系统来了解。

Step 1: The attacker initiates the attack by sending the following value as the value of the token
token=a

步骤 1：攻击者初始可以发送 `token=a` 值启动攻击

The request will look like this

具体请求如下

```
GET /stuff/form.php?id=token=a
```

The response will be as follows

返回响应如下

```
<a href="form2.php?token=csvfdfcrvet343v">Go to form2</a>
<form target="https://example.com:443/stuff/everything.php?id=token=a">
…………
```

The length of the response will decrease by the 6 viz. the value of the length of the following string ‘token=’.

响应的长度将会减少6，长度为字符串 `token=a` 的长度。

Step 2: The attacker looks at the response length and concludes that the value of the token is incorrect.

步骤 2：攻击者查看响应并得出 `token` 不正确的结论

The attacker repeats the process with different value of ‘token’.

攻击者使用不同的 `token` 值重复进行以上操作

Step 3:  When the attacker tries ‘token=c’, the request looks like this:

步骤 2：当攻击者尝试使用 `token=c`，请求如下

```
GET /stuff/form.php?id=token=c
```

The following will be the response

返回响应如下

```
<a href="form2.php?token=csvfdfcrvet343v">Go to form2</a>
<form target="https://example.com:443/stuff/everything.php?id=token=c">
…………
```

Since the first character of the attacker’s guess matches the actual value of the token the response length decreases by 7.

攻击者猜测的第一个字符与 `token` 实际第一个值匹配，因此响应长度减少7。

Step 4: The attacker concludes that he has successfully guessed the first character of the token. The attacker then keeps the first character constant and varies the second character and repeats the entire process again.

步骤 4：攻击者得出结论他已成功猜到了 `token` 的第一个字符。然后攻击者保持第一个字符不变并改变第二个字符然后再次重复整个过程。

Step 5: The attacker does this until he has successfully guessed the entire value of the ‘token’.

步骤 4：攻击者执行此操作直到他成功猜到 `token` 的全部值。


## Impact of BREACH attack (BREACH 攻击的影响)

The BREACH attack can be practically executed under a minute depending on number of per thousand requests required as per the secret size. The power of the attack comes from the fact that it allows guessing a secret one character at a time.

BREACH 攻击根据机密数据大小构造的数千请求实际上可以在一分钟内执行完成。攻击的强力之处来源于一次猜测一个字符。


## Mitigations for BREACH attack (BREACH 攻击防范措施)

The following mitigations must be applied for BREACH attack

对于 BREACH 攻击可以采用以下缓解措施

Disabling HTTP compression
Separating secrets from user input
Randomizing secrets per request
Masking secrets (effectively randomizing by XOR’g with a random secret per request)
Length hiding (by adding random amount of bytes to the responses)
Rate-limiting the request

- 禁用HTTP压缩
- 用户的输入剥离机密数据
- 机密数据每次请求随机产生
- 屏蔽机密数据（每个请求讲加密数据进行异或会非常有效）
- 限制请求的速率


## Conclusion (结论)

BEAST, CRIME and BREACH are a part of the increasing number of attacks on the SSL/TLS protocol. It is important that we should remain aware of them and apply the recommended mitigation measures to secure our data flowing through SSL.

BEAST, CRIME 与 BREACH 在 SSL/TLS 协议攻击中出现的次数越来来越多。重中之重的是我们要意识到他的重要并且使用缓解措施保证使用 SSL 的数据安全。

## 原文

[SSL/TLS attacks: Part 3 – BREACH Attack](http://niiconsulting.com/checkmate/2013/12/ssltls-attacks-part-3-breach-attack/)

****
**THE END [ 2019-01-21 ]**