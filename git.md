# 自建GIT

决定搭建git后，对开源的webgit做了下调研，最后决定在gitlab和gitea中选一个。

- gitlab： 对标github，生态比较全，自带功能丰富。自带静态页面渲染，自带CI/CD。但是gitlab太重了，镜像拉下来接近1个G，k8s部署要起n个svc。
- [gitea](https://gitea.io/zh-cn/)： 新兴的gitserver，据说是国人写的。小巧轻便，运行开销小，但是CI/CD功能没有gitlab全。

考虑到我的云服务器只有2c8g,50G硬盘空间，最终还是决定部署gitea。如果需要CI/CD功能，正好也可以试验下和其他开源产品的对接。

## 部署 GITEA

不~翻墙~我家联通网居然打不开gitea首页，差评。 

gitea官方提供了各种安装方式。但作为坚定的云原生一员，肯定是docker 或者k8s二选一。

- docker简单方便， docker pull docker run就ok了。

- k8s稍微复杂些，不过官方也给了helm chart，可以一键部署。

纠结了一下，最红还是决定用k8s。一是本身打算转行SRE，想考CKA、CKD，提前熟悉环境肯定是有必要的，边玩边学才是最有效率的。二是我打算用K3s，这样服务器资源占用率也不会很高。

### 准备工作

参考文档： chart gitea doc

