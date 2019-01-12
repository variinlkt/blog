# 前端工具使用1-Charles使用记录
记录一些Charles常用的功能
## 抓包
- 关闭其他代理软件
- Charles -> Proxy -> 勾选Macos Proxy
- Help -> SSL Proxying -> install Charles root certificate -> 钥匙串访问信任charles证书

### https抓包
![](https://upload-images.jianshu.io/upload_images/1901430-ae1660183e196ded.png)
![](https://upload-images.jianshu.io/upload_images/1901430-0a18a221d862166a.png)
- 手机应与电脑在同一wifi下，设置代理，然后访问网址并下载证书，信任证书
- Proxy -> SSL Proxying Settings
![](https://upload-images.jianshu.io/upload_images/1901430-1d6e60dbb846cb29.png)

#### 原理简析
如果是HTTP请求，因为数据本身并没加密所以请求内容和返回结果是直接展现出来的。
但HTTPS是对数据进行了加密处理的，如果不做任何应对是无法获取其中内容。所以Charles做的就是对客户端把自己伪装成服务器，对服务器把自己伪装成客户端：

Charles拦截客户端的请求，伪装成客户端向服务器进行请求
服务器向“客户端”（实际上是Charles）返回服务器的CA证书
Charles拦截服务器的响应，获取服务器证书公钥，然后自己制作一张证书，将服务器证书替换后发送给客户端。（这一步，Charles拿到了服务器证书的公钥）

客户端接收到“服务器”（实际上是Charles）的证书后，生成一个对称密钥，用Charles的公钥加密，发送给“服务器”（Charles）
Charles拦截客户端的响应，用自己的私钥解密对称密钥，然后用服务器证书公钥加密，发送给服务器。（这一步，Charles拿到了对称密钥）
服务器用自己的私钥解密对称密钥，向“客户端”（Charles）发送响应
Charles拦截服务器的响应，替换成自己的证书后发送给客户端

当然，如果用户不选择信任安装Charles的CA证书，Charles也无法获取请求内容。还有一种，如果客户端内置了本身的CA证书，这时如果Charles把自己的证书发送给客户端，客户端会发现与程序内的证书不一致，不予通过，此时Charles也是无法获取信息的。

## 映射
- Charles -> Tools -> Map Local或右键某个请求选择Map Local
- 然后选择一个本地的文件就ok

## 请求接口返回数据修改
- Charles -> Tools -> Rewrite
- 勾选`enable rewrite`
- 添加规则
- 
![](https://upload-images.jianshu.io/upload_images/1392062-bc94af091927aefc.png)

![](https://upload-images.jianshu.io/upload_images/1392062-74377fbc5908d235.png)

```
{"status":"failed","code":400,"desc":"invalid argument"}
```
![](https://upload-images.jianshu.io/upload_images/1392062-d4f6f61b741439db.png)

###### 相关链接
https://www.jianshu.com/p/276bb5a49e59

https://www.jianshu.com/p/ec0a38d9a8cf
