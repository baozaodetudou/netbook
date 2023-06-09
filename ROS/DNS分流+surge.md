
### 首先要知道的一些名词解释（来源网络不正确请指正）

> 什么是 Fake IP 
- 当用户发送 DNS 请求时，代理并不发送请求到远程 DNS 服务器，而是为每个域名返回一个唯一假 IP。
- 在指向这些假 IP 的流量发送到代理时，按查询得到的域名重置当前连接的目标。

> Surge

Surge 是为开发人员设计的 Web 开发和代理实用程序，因此需要专业知识才能使用。
Surge 的核心工作流程包括四个主要功能：
- 接管：Surge 允许用户接管设备发送的网络连接。支持代理服务和虚拟网卡接管。
- 处理：该软件使用户能够修改已被接管的网络请求和响应。这包括 URL 重定向、本地文件映射、使用 JavaScript 的自定义修改以及许多其他方法。
- 转发：网络请求一旦被接管，就可以由用户转发到其他代理服务器。转发可以是全局的，也可以使用灵活的规则系统来确定出站策略。
- 拦截：用户可以截获和保存来自网络请求和响应的特定数据。此外，用户还可以使用 MITM 解密 HTTPS 流量。


## 说明

> 此设置是本人通过大佬的手把手教，设置家里网络时候总结的仅供参考

本次设置是routerOS以后都简称ROS+Surge来搭建的家庭网络环境；简单理解就是我们平常经常流行的爱快+OP的模式；
主路由是ROS,负责拨号上网和DNS解析（有点不一样下边说明）；MacOS安装surge来作为旁路由进行流畅的git访问（节点请自备）;

> 如果你接触过ROS,下边的文字基本上会理解他的原理如果不理解可以看看下边的流程图

![DNS分流.png](./DNS分流.png)

```jupyter

假设我有设备A和B, A配置了gfw的标签(这个标签是用来告诉ROS要走surge)；B没配置；
A和B分别访问goole.com简称G(google.com在静态DNS里) 和 baidu.com简称D；
A+G就会匹配静态DNS的fakeip请求去surge请求; 
A+D就直接匹配不到然后走真实ip;
B+G也是会匹配的fakeip但是由于没标签就不会走surge只会走ROS，因为不是DNS解析的DNS肯定访问不到（优：防止DNS,缺点：不能随便配置静态DNS）；
B+D会通过DNS来解析真实ip，ROS直接可以连接
```

### 脚本分享
> 脚本来源于会所群友
- 下边脚本如果不好复制，可以查看请查看文件 [ros_dns.conf](./ros_dns.conf) 
- surge相关的配置文件在surge下可以获取到 [surge.conf](../surge/surge.conf) [配置中的xxxx请自行更换成自己对应的配置]

 ```shell
## 假定surge gateway的地址是192.168.20.7，需要梯子的内网设备ip是192.168.20.73 也可以使用ip段192.168.20.73-192.168.20.77

# 添加table表
/routing table add name=gfw fib

# 添加需要走梯子的内网设备ip
/ip firewall address-list add address=192.168.20.73 list=proxy


# 打标记
/ip firewall mangle add chain=prerouting src-address=192.168.20.7 action=accept
/ip firewall mangle add chain=prerouting src-address-list=proxy dst-address=198.18.0.0/16 dst-address-type=!local action=mark-routing new-routing-mark=gfw

# 打gfw标记的走surge
/ip route add dst-address=198.18.0.0/30 gateway=192.168.20.7 routing-table=main
/ip route add dst-address=0.0.0.0/0 gateway=192.168.20.7 routing-table=gfw

# 添加静态dns
/ip dns set cache-size=20560KiB

# 添加校长的静态dns list
/ip dns static
add comment=KEYWORD forward-to=198.18.0.2 regexp=".*(\\.)\?(google|facebook|bl\
    ogspot|jav|pinterest|pron|github|bbcfmt|uk-live|hbo).*" ttl=1h type=FWD
add comment=KEYWORD forward-to=198.18.0.2 regexp=".*(\\.)\?(dropbox|hbo).*" \
    ttl=1h type=FWD
add comment="Public CDN" forward-to=198.18.0.2 regexp=".*(\\.)\?(akamaized|aka\
    maihd|cloudfront|cloudflare|hinet|alphacdn|fastly).*" ttl=1h type=FWD
add comment="Apple Services" forward-to=198.18.0.2 regexp=\
    ".*\\.(icloud|me)\\.com\$" ttl=1h type=FWD
add comment="Apple Services" forward-to=198.18.0.2 regexp=".*(\\.)\?(appsto|ap\
    pstore|aaplimg|crashlytics|mzstatic).*(\\.com|\\.co|.re)" ttl=1h type=FWD
add comment=Amazon forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(amazon|amazonaws|kindle|primevideo).*\\.com" ttl=1h type=FWD
add comment=MicroSoft forward-to=198.18.0.2 regexp=".*(\\.)\?(azure|bing|live|\
    outlook|msn|surface|sharepoint)\\.(net|com|org)" ttl=1h type=FWD
add comment=Hulu forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(hulu|happyon).*(\\.com|\\.jp)" ttl=1h type=FWD
add comment=Pornhub forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(pornhub|phncdn).*\\.com" ttl=1h type=FWD
add comment=Netflix forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(nflx|netflix|fast).*(\\.net|\\.com)" ttl=1h type=FWD
add comment=BBC forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(bbc|bbci)\\.(co\\.uk|com)" ttl=1h type=FWD
add comment=AbemaTV forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(abema|ameba|hayabusa)\\.(jp|io)" ttl=1h type=FWD
add comment=Speedtest disabled=yes forward-to=119.29.29.29 regexp=\
    ".*(\\.)\?(speedtest|hgc)\\.(net|com)" ttl=1h type=FWD
add comment="IP INFO" forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(ipinfo|ip)\\.(io|sb)" ttl=1h type=FWD
add comment=encoreTVB forward-to=198.18.0.2 regexp="(edge\\.api\\.brightcove|v\
    ideos-f\\.jwpsrv|content\\.jwplatform)\\.(com|net)" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(youtube|ytimg)\\.(com)" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(android|androidify|appspot|autodraw|blogger)\\.com" ttl=1h \
    type=FWD
add comment=Google forward-to=198.18.0.2 regexp=".*(\\.)\?(capitalg|chrome|chr\
    omeexperiments|chromestatus|creativelab5)\\.com" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(debug|deepmind|dialogflow|firebaseio|googletagmanager)\\.com" \
    ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(ggpht|gmail|gmail|gmodules|gstatic|gv|gvt0|gvt1|gvt3)\\.com" \
    ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(itasoftware|madewithcode|synergyse|tiltbrush|waymo)\\.com" \
    ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(widevine|x|xn--ngstr-lra8j)\\.(company|com)" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(ampproject|certificate-transparency|chromium)\\.org" ttl=1h \
    type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(getoutline|godoc|golang|gwtproject)\\.org" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(polymer-project|tensorflow)\\.org" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(material|shattered|recaptcha)\\.(net|io)" ttl=1h type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(waveprotocol|webmproject|webrtc|whatbrowser)\\.org" ttl=1h \
    type=FWD
add comment=Google forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(abc|admin|getmdl)\\.(xyz|net|io)" ttl=1h type=FWD
add comment=Facebook forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(messenger|whatsapp|oculus|oculuscdn)\\.(com|net)" ttl=1h type=\
    FWD
add comment=Facebook forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(cdninstagram|fb|fbcdn|instagram)\\.(com|net|me)" ttl=1h type=\
    FWD
add comment=Twitter forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(twimg|twitpic|twitter)\\.(co|com)" ttl=1h type=FWD
add comment=Line forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(line(.*|\\.)|naver)\\.(me|com|net|jp)" ttl=1h type=FWD
add comment=Get forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(medium|wikipedia)\\.(com|org)" ttl=1h type=FWD
add comment=Community forward-to=198.18.0.2 regexp=".*(\\.)\?(jkforum|520cc|st\
    eamcommunity|reddit|redditmedia|v2ex|hostloc)\\.com" ttl=1h type=FWD
add comment=Community forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(mobile01)\\.com" ttl=1h type=FWD
add comment=Video/Pic forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(tumblr|vimeo|flickr|vine|pinimg)\\.com" ttl=1h type=FWD
add comment="Android APK" forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(apk-dl|apkpure)\\.com" ttl=1h type=FWD
add comment=XXX forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(t66y|xvideos|pronhub|avgle)\\.com" ttl=1h type=FWD
add comment=Telegram forward-to=198.18.0.2 regexp=".*(\\.)\?telegram\\.org" \
    ttl=1h type=FWD
add comment=Tools forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(shadowsocks|v2ray|putty|fixit)\\.(org|com)" ttl=1h type=FWD
add comment=VPS forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(bandwagonhost|bwh1|vultr|digitalocean|linode|feenom)\\.com\$" \
    ttl=1h type=FWD
add comment=PT forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(pterclub|beitai|hd4fans|m-team|chdbits|ourbits|hdchina).*" \
    ttl=1h type=FWD
add comment=PT forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(hdsky|pterclub|totheglory).*" ttl=1h type=FWD
add comment=PT forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(yingk|springsunday|keepfrds).*" ttl=1h type=FWD
add comment=Disney+ forward-to=198.18.0.2 regexp=\
    ".*(\\.)\?(dssott|disneyplus|disney-plus|bamgrid)\\.(com|net)" ttl=1h \
    type=FWD
add comment=Disney+ forward-to=198.18.0.2 name=cdn.registerdisney.go.com ttl=\
    1h type=FWD
add comment=Plex forward-to=198.18.0.2 regexp=".*(\\.)\?(plex)\\.tv" ttl=1h \
    type=FWD


```