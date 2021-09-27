---
title: '更换 AlphaSSL 单域名证书加速国内 OCSP 认证遇到的坑'
date: '2018-04-11'
tags: [主机壳, Netlify, SSL]
draft: false
summary: '最近决定更换一下我的博客大陆访问时候所用的 SSL 证书，于是到 [SSL2BUY](https://www.ssl2buy.com/) 上买了一个 AlphaSSL 的单域名证书，然而这才是坑的开始。因为按照最初的设置，这货弄出来只支持 WWW Domain, 显然这是不科学的。'
---

最近决定更换一下我的博客大陆访问时候所用的 SSL 证书，于是到 [SSL2BUY](https://www.ssl2buy.com/) 上买了一个 AlphaSSL 的单域名证书，然而这才是坑的开始。因为按照最初的设置，这货弄出来只支持 WWW Domain, 显然这是不科学的。

## 更换背景

### 原因及分析

起因是最近一次测试国内访问速度的时候，发现原来使用的 `COMODO ECC` 单域名证书在进行在线证书状态协议 `OCSP` 这种阻断式认证的时候，延时在某些情况下可能会达到 2s. 这简直就是一件不能忍的事情。

`COMODO ECC` 的 `OCSP` 校验地址是 [http://ocsp.comodoca.com](http://ocsp.comodoca.com), 感兴趣的话可以测试一下延迟，国内应该是解析到一个英国的 IP 上。另外附上`COMODO ECC` 的 `CRL` 证书撤销列表地址：[http://crl.comodoca.com/COMODOECCDomainValidationSecureServerCA.crl](http://crl.comodoca.com/COMODOECCDomainValidationSecureServerCA.crl)

然而我国内访问所使用的主机供应商[主机壳](https://i.zhujike.com/flag/5382)并不能开启 `OCSP Stapling` 来避免这一延迟。因此只有采取更换有更快 `OCSP` 访问速度的证书来改善访问体验。

### 为什么是 `AlphaSSL`

这个很简单， `AlphaSSL` 的 `OCSP` 校验地址是 [http://ocsp2.globalsign.com/gsalphasha2g2](http://ocsp2.globalsign.com/gsalphasha2g2), 这个地址通过测试会发现大陆有众多访问节点，延迟极低。采用同样服务器的还有 `GlobalSign` 的证书，毕竟是一家，但由于定位问题，价格肯定是 AlphaSSL 便宜。

什么？为什么不申请那些由国内 CA 签发的免费证书，比如[数安时代](https://www.trustauth.cn/free-ssl)? 还是信任问题，WoSign 事件早就把国内 CA 的信用败光了。况且看了下数安的申请流程，还要联系客服什么的，真是麻烦。

### 为什么不选择 AlphaSSL WildCard 而选择单域名证书

市面上的 AlphaSSL WildCard 除了正规购买的，就是通过滥用某主机商福利生成的。后者虽然免费，但风险不低，随时可能被 Revoke, 且有违道德。

再一个就是我目前没有泛域名使用需求，购买价格高昂的泛域名实属没事找事。

选择 [SSL2BUY](https://www.ssl2buy.com/) 纯粹是因为其价格和口碑，比淘宝上口碑特别好的代理商价格还便宜。

## 购买流程

这个似乎没有什么好写的，下单，付款，提交 CSR, 进行 DNS 认证然后就能收到证书。

## 坑及解决方案

### 生成的证书仅支持 WWW 域名而不支持 non-WWW 域名

最开始，我生成的证书仅包括了 [www.woadzs.me](www.woadzs.me) 这个 WWW 域名而不包含 [woadzs.me](woadzs.me) 这个裸域。这显然是不正常的。后来发现这是标准证书申请的一个坑，泛域名压根儿碰不到这种情况。

根据 [Domain Verification Changes](https://support.globalsign.com/customer/en/portal/articles/2644394-domain-verification-changes) 提供的信息，按照 GlobalSign 的规定为了 SSL 同时支持 WWW 和 non-WWW, 正确的操作应该是**提交以 www.example.com 为 Common Name 的 CSR. 在认证的时候，无论是通过 URL 还是 DNS TXT 记录认证，都必须在 example.com 下进行域名所有权验证。**

正确的操作只有上面一条，手抖把 non-WWW 写成了 CN 或者是使用 WWW 进行了验证或者是其他排列组合，都只能收到“残废”证书。

### 在线补全证书链不正常

我们都有拿到新证书先补全证书链的习惯，而我们也似乎都习惯于使用在线的证书补全工具，比如 [MySSL.com 证书链修复](https://myssl.com/chain_download.html) 和 [certificatechain.io](https://certificatechain.io/).

然而这些在线证书链在补全 `AlphaSSL Standard Certificate` 的时候，所提供的中级证书 `Intermediate Certificates` 并不是用来签署我这个新证书的那一个。

**原因是 GlobalSign 和 AlphaSSL 为 2014 年 3 月 31 号之后签署的新 DV 证书启用了新的中级证书，而网上自动补全的统统是给补老中级证书。**新老中级证书快速判断的标准就是有效期：

- 新中级证书有效期为 20 February 2024
- 老中级证书有效期为 02 August 2022

适用于 AlphaSSL 的新老中级证书，除了在证书签发的通知邮箱里面会告知具体链接，也可以在 [Install Root Certificate](https://www.alphassl.com/support/install-root-certificate.html) 找到。

PS: 无论是 AlphaSSL 还是 Let's Encrypt, 现在的中级证书都是基于 RSA 2048 bits 的， ECC 的中级证书还需要等等。Let's Encrypt 给出的[时间点](https://letsencrypt.org/upcoming-features/)是 2018 年 Q3 ~~然而已经跳票多次~~. 而 AlphaSSL 没有给出具体时间节点。

## 对主机壳和 Netlify 的碎碎念

- 求开通 HSTS 功能 ← 本条仅针对主机壳
- 求开通 OCSP 装订配置
- 求开通 ECC 证书支持 ← 本条仅针对 Netlify

## 后记——目前（2018 年 4 月）网站的 SSL 配置及虚拟空间特性

### 大陆访问（[主机壳](https://i.zhujike.com/flag/5382)）

- AlphaSSL ECDSA 256 bits
- 全站 HTTPS
- 全站 HTTP/2
- 静态资源访问大陆 CDN
- 网页本体访问香港 CDN
- 详细证书信息可查看 [MySSL.com](https://myssl.com/woadzs.me?domain=woadzs.me)

### 海外访问（[Netlify](https://www.netlify.com/)）

- Let's Encrypt RSA 2048 bits
- 全站 HTTPS 且加入 HSTS 标记
- 全站 HTTP/2
- 全站访问 Amazon CloudFront CDN
- 详细证书信息可查看 [SSLLabs](https://www.ssllabs.com/ssltest/analyze.html?d=woadzs.me)
