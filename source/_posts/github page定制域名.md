---
title: github page定制域名
date: 2022-11-12T17:31:31+08:00
tags: [domain, 'github page']
categories: blog
references:
     -
          title: 网站备案概述
          url: https://cloud.tencent.com/document/product/243/18907
     -
          title: 域名隐私保护
          url: https://cloud.tencent.com/document/product/242/30404
     -
          title: HTTP Cookie
          url: https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies
     -
          title: Managing a custom domain for your GitHub Pages site
          url: https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/managing-a-custom-domain-for-your-github-pages-site
     -
          title: Verifying your custom domain for GitHub Pages
          url: https://docs.github.com/cn/pages/configuring-a-custom-domain-for-your-github-pages-site/verifying-your-custom-domain-for-github-pages
---

本文主要讲解为github page搭建的博客申请域名的过程，以及在这个过程中的一些经验之谈。本文主要涉及到以下几个方面：
* 域名注册商的选择方法
* 域名申请过程中是否需要实名认证、是否需要进行ICP备案
* 如何为github page博客配置DNS记录，以便通过新域名可以访问github page

## 域名申请
所谓的域名申请（购买），并不意味着你能够永久地拥有域名，实际上只能够租用一定期限的域名。用户一般通过域名注册商(registrar)/域名分销商(reseller)申请域名。域名一般是付费申请，不同的域名注册商/分销商对同一域名的定价、续费不同。也存在免费的域名注册商[Freenom](https://www.freenom.com/zh/freeandpaiddomains.html)提供`.TK`、`.ML`、`.GA`、`.CF`、`.GQ`顶级域名的注册。如果要选择心仪的TLD，还是选择付费域名进行注册。

选择域名注册商主要考虑三个维度：
* 价格
* 隐私保护
* 注册商服务

用户选购域名时，需要不仅考虑当前的注册价格还需要考虑后期续费价格。很多域名注册商会提供域名注册优惠，但往往只能注册1年，注册价格极低，但后期续费价格较高。我对比了国内腾讯云、阿里万网以及国外的几家域名注册商，最终了选择了国内的域名注册商，国外的域名注册商如Namesilo的续费价格较高。

### 隐私保护
可能你经常听到注册域名要提交身份信息、对网站要进行备案，实际上真的如此吗？

我国境内的域名注册服务商、域名注册管理机构遵守[工信部2017年《中国互联网络域名管理办法》](http://www.cac.gov.cn/2017-09/28/c_1121737753.htm)的域名实名认证的要求：
> 域名注册申请者应提交域名持有者真实、准确、完整的域名注册信息进行实名认证，若域名在规定时间内未通过实名审核，会被注册局暂停解析(Serverhold)。对于不符合规定的域名，将依法予以注销。

因此，**选择国内域名注册服务商需要提交身份证信息进行实名认证**。

根据国务院令第292号[《互联网信息服务管理办法》](https://cloud.tencent.com/document/product/243/50316)和[《非经营性互联网信息服务备案管理办法》](https://cloud.tencent.com/document/product/243/50315)规定，国家对经营性互联网信息服务实行许可制度，对非经营性互联网信息服务实行备案制度。未获取许可或者未履行备案手续的，不得从事互联网信息服务，否则属于违法行为[1]:
> 第二条 在中华人民共和国境内从事互联网信息服务活动，必须遵守本办法。
>
> 本办法所称互联网信息服务，是指通过互联网向上网用户提供信息的服务活动。

因此，使用中国大陆境内的服务器提供web服务必须进行ICP备案，像github page服务器在国外，则不需要进行ICP备案。如果用户的网站服务器在中国大陆，**建议（非必须）选择国内的域名注册商**，如阿里云、腾讯云，他们有辅助ICP备案的流程，方便用户进行备案。当然，用户也可以选择国外域名注册商，只是ICP备案会麻烦一点。

通常，无需担心实名认证的身份信息被泄露。根据 ICANN 《通用顶级域名注册数据临时政策细则（Temporary Specification for gTLD Registration Data）》和欧盟《通用数据保护条例》合规要求，域名注册商保证whois查询结果中域名管理人员的相关个人数据，包括姓名、邮箱、电话、街道地址等。

但国内某些注册商并不能完全做到这一点，我在腾讯云注册的`.fun`域名通过whois查询到的`Registrant Organization`字段包含注册人的英文名（默认为姓名拼音）。v2ex的网友对这个情况也有[讨论](https://www.v2ex.com/t/846393)，腾讯云对此答复：
> 通过腾讯云注册的域名，除 .com / .net 等 Verisign 域名的 WHOIS 信息由腾讯云直接按调整后的规则提供外，其他腾讯云域名的 WHOIS 信息均由相应的注册局提供，并由各注册商自行决定在其 WHOIS 平台的显示信息内容。由于各注册局、注册商对于 GDPR 和 WHOIS 显示信息调整的落实方案与进度暂不统一，因此相关域名在第三方 WHOIS 平台如何显示，取决于对应的注册局和注册商政策。
通过腾讯云注册的 .com / .net 等 Verisign 域名，目前通过第三方 WHOIS 平台所能查询到的腾讯云域名注册联系人信息均为缓存信息，最新查询结果中，已不再包括腾讯云域名的注册联系人信息。[2]

建议在国内注册域名的朋友，填写英文名时特意做好规避。如果要检查你的注册信息是否能被whois查询到，建议通过以下两个网站进行查询：
* https://whois.chinaz.com/
* https://whois.gandi.net/

国内域名注册商的whois查询服务确实隐藏了个人信息。即便隐藏了英文名，仍然会泄露注册人所在的省份。

### 注册商服务
选择域名注册商也需要考虑注册商服务，这里简单罗列下：
* 域名转入/转出限制：有的域名注册服务商对于域名转入/转出有较为严格的限制，影响用户体验，可以事先进行调研
* 支付方式：若选择国外域名服务商，则需要考虑支付的便捷性（是否支持支付宝/微信支付），国外几家比较大的域名注册商如namesilo、name.com、GoDaddy都支持支付宝
* SSL证书申请：如果网站需要支持https，域名服务商能免费支持SSL证书申请则会相对便捷，github page本身会提供https服务，可忽略这点

当然，根据自身的实际需求，对注册商的服务要求也大不相同，欢迎在评论区交流。

## github page关联域名
github page支持设置的自定义域类型：

| 自定义域类型 | 范例  |
| -- | -- |
| www子域(subdomain) | `www.justhack.fun` |
| 自定义子域 | `blog.justhack.fun` |
| 顶点域(apex domain) | `justhack.com` |

Github建议同时配置顶点域和www子域：
> When using an apex domain, we recommend configuring your GitHub Pages site to host content at both the apex domain and that domain's www subdomain variant.

### apex domain vs www subdomain
`www`子域能让人一眼看出域名是一个浏览器可以打开的URL。直接采用顶点域的方式可以让域名更加简洁，如github、twitter都采用顶点域。Github也支持github page仅设置顶点域/www子域/自定义子域。

使用顶点域存在"cookie污染"的问题：即顶点域的cookie作用范围包括其所有子域。即访问`justhack.fun`产生的cookie，在访问`foo.justhack.fun`或者`bar.justhack.fun`都会带上。对访问的安全、性能都有影响。

HTTP Cookie（也叫 Web Cookie 或浏览器 Cookie）是服务器发送到用户浏览器并保存在本地的一小块数据。浏览器会存储 cookie 并在下次向同一服务器再发起请求时携带并发送到服务器上。通常，它用于告知服务端两个请求是否来自同一浏览器——如保持用户的登录状态。Cookie 使基于无状态的 HTTP 协议记录稳定的状态信息成为了可能。[3]

cookie主要用在三个方面[3]：
* 会话状态管理: 如用户登录状态、购物车、游戏分数或其它需要记录的信息
* 个性化设置: 如用户自定义设置、主题和其他设置
* 浏览器行为跟踪: 如跟踪分析用户行为等

服务器收到 HTTP 请求后，服务器可以在响应标头里面添加一个或多个 `Set-Cookie`选项。浏览器收到响应后通常会保存下Cookie，并将其放在 HTTP Cookie 标头内，向同一服务器发出请求时一起发送。http响应头中的`Set-Cookie`格式如下：
```html
Set-Cookie: <cookie-name>=<cookie-value>
```
下图是用户通过浏览器请求`www.example.com`传输cookie示例：
![](cookie.png)

1. 浏览器作为用户代理发送https请求访问`www.example.com`
2. 服务器告知客户端存储一对cookie：`foo_cookie`、`bar_cookie`。
3. 接着访问`www.example.com`域名下的`sample_page.html`页面会带上两个cookie。

可以通过两种方式确定访问域名是否带有cookie：
1. 在浏览器(以chrome)为例，按`F12`进入开发者工具 -> 点击`Application/Cookies`即可查看相应域名的Cookie，可见`justhack.fun`并不包含任何cookie
![](chrome_cookie.png)
2. 执行`curl -I https://justhack.fun`命令，该命令仅返回http(s)头部信息，但不返回页面内容，检查头部信息中的`set-cookie`字段，同样发现访问`justhack.fun`并不包含任何cookie：
```bash
❯ curl -I https://justhack.fun
HTTP/1.1 200 Connection established

HTTP/2 200 
server: GitHub.com
content-type: text/html; charset=utf-8
last-modified: Sat, 12 Nov 2022 16:57:25 GMT
access-control-allow-origin: *
etag: "636fd075-207c0"
expires: Mon, 14 Nov 2022 05:24:54 GMT
cache-control: max-age=600
x-proxy-cache: MISS
x-github-request-id: BA4C:4119:22EE90:2983D9:6371CECE
accept-ranges: bytes
date: Mon, 14 Nov 2022 05:14:55 GMT
via: 1.1 varnish
age: 0
x-served-by: cache-hkg17935-HKG
x-cache: MISS
x-cache-hits: 0
x-timer: S1668402895.753872,VS0,VE260
vary: Accept-Encoding
x-fastly-request-id: a144f45239faaabca10c759cae386e23e310938b
content-length: 133056
```
因为github page是静态页面，不带任何cookie，所以github page建议同时设置apex domain和www subdomain的方式既能便于访问，同时也不存在cookie泄露的风险。

### 配置apex domain和www subdomain
如果github page配置了`www.justhack.fun`作为域名，且配置了正确的www子域和apex域的DNS记录，则访问`justhack.fun`时会重定向到`www.justhack.fun`。自动重定向仅适用于www子域，其他子域不适用。下面介绍如何为github page同时配置www subdomain和apex domain：

1. 在github page代码仓中配置域名，以我本人的github page为例：打开[github page代码仓](https://github.com/system-thoughts/system-thoughts.github.io) -> 点击settings -> 点击“Code and automation/Pages”，在"Custom domain"处输入域名，此处输入www subdomain：`www.justhack.fun`,也可以输入apex domain:`justhack.fun`。这个动作会在github page的发布分支生成一个"Update CNAME"的提交，由于我的github page发布分支是`gh-pages`分支，则会在该分支生提交。我是通过travis-ci自动部署博客内容，每当更新master分支，travis-ci会覆盖更新gh-pages分支，该分支只有一次提交。需要在master分支的`source`目录下手动添加`CNAME`文件，文件中仅包含`www.justhack.fun`。如果没有此步操作，后续更新master分支，访问域名会404。
2. 对apex domain和www subdomain添加domain record:
    * apex domain：添加A记录或者AAAA记录，apex domain不建议添加CNAME记录，虽然部分DNS provider并未对此做了严格限制，但该行为极度不推荐。
    ```
    A record:
    185.199.108.153
    185.199.109.153
    185.199.110.153
    185.199.111.153

    AAAA record:
    2606:50c0:8000::153
    2606:50c0:8001::153
    2606:50c0:8002::153
    2606:50c0:8003::153 
    ```
    * www subdomain: 添加CNAME记录，指向github page的`github.io`域名。

出于安全考虑，需要先执行完步骤1再执行步骤2。先配置DNS记录，再配置github page的域名会导致其他github用户的github page使用你的一个子域名[4]
{% noteblock warning %}
Make sure you add your custom domain to your GitHub Pages site before configuring your custom domain with your DNS provider. Configuring your custom domain with your DNS provider without adding your custom domain to GitHub could result in someone else being able to host a site on one of your subdomains.
{% endnoteblock %} 

3. 添加DNS记录成功之后，步骤1中"Custom domain"下方的“DNS check”会成功，点击下面的"Enforce HTTPS"让github page支持https。
4. 域验证(verify a custom domain)，进行域验证能够阻止其他github用户接管您的自定义域，用其发布自己的github page。删除你的github page代码仓、github付费服务被取消等场景会出现这种情况。验证域会包含其任何直接子域。 例如，如果已验证 github.com 自定义域，docs.github.com、support.github.com 和任何其他直接子域也将受到保护，以防止被接管[5]。具体步骤如下：
   1. 点击github个人资料照片 -> 选择settings
   2. 点击"Code, planning, and automation/Pages" -> 添加"Add domain"填写apex domain
   3. 会生成一条DNS TXT记录，按照提示，在DNS provider中添加该DNS TXT记录
   4. 可以退出之后，再进入该页面，查看域名是否已经验证

经过上述步骤后，已经完成了github page自定义域名。当前可以通过www subdomain和apex domain访问github page。步骤1配置github page域名是www subdomain，这里表示通过www subdomain访问github page能够直接获取到页面内容；通过apex domain访问github page会进行301重定向。
```bash
❯ curl  https://justhack.fun
<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

使用301跳转能够帮助SEO优化，因为www subdomain和apex domain实际上访问的是同一网站资源，如果不进行apex domain到www subdomain的301跳转，搜索引擎会认为这是两个不同的域名，导致域名的权重分散，不利于SEO排名。
可以将github page的域名配置为apex domain，则是www subdomain 301跳转至apex domain。我个人更加倾向将github page域名配置为www subdomain，因为其会产生一条CNAME记录，这和配置www subdomain的DNS记录一致。