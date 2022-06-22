---
layout: post
title: 'LDAP 集成之 Dokuwiki 篇'
subtitle: '利用 LDAP 为 Dokuwiki 提供用户认证服务'
date: 2022-06-22 15:40:00 +0900
categories: [tech, webmaster]
author: zhonger
cover: 'https://i.luish.cc/cover/0QQ3x6.webp'
cover_author: 'Artur Voznenko'
cover_author_link: 'https://unsplash.com/@voznenko_artur'
tags:  
- LDAP
- Dokuwiki
---

## 前言

### 百科的发展

&emsp;&emsp;说到百科，国际上最出名的莫过于 [WikiPedia](https://wikipedia.org)，而国内最出名的莫过于[百度百科](https://baike.baidu.com)。当然，这两者也有一些差别。WikiPedia 支持多种语言，可以自由编辑，不过一般会要求提供必要的参考资料及链接来佐证。百度百科则只支持中文，对于编辑条目也有比较高的要求，一般来说是由专门的人员编写、审核。除了这两家之外，也有一些其他的大众百科，比如 [中国大百科](https://www.zgbk.com)、[360 百科](https://baike.so.com)、[搜狗百科](https://baike.sogou.com) 等。大众百科的显著特点是范围非常广，适合大众科普，而想要查找一些太过详细的知识可能无法满足。

&emsp;&emsp;于是随着百科特殊化的需求增大，越来越多的专门化或者特殊化的百科也开始涌现，比如与计算机专业相关的 [集智百科](https://wiki.swarma.org)、[AI 知识库](https://easyai.tech)，与游戏相关的 [灰机 Wiki](https://www.huijiwiki.com/wiki)，与二次元动漫、小说相关的 [萌娘百科](https://zh.moegirl.org.cn) 等等。这在一定程度上弥补了大众百科的不足，满足了大众对于专门化知识科普或者手册的需求。

### 开源百科程序

&emsp;&emsp;既然是百科站点，就需要有百科程序来支撑用户管理、条目编辑、条目审核等功能。其实，世界上最大的百科站点 WikiPedia 使用的是免费开源的 [MediaWiki](https://www.mediawiki.org/)。而百度百科则是采用自家开发的闭源程序，且与百度账号、百度知道等百度系产品打通。如果自己想要搭建一个百科站点，除了 MediaWiki 外，还有很多免费的选择，比如 [Dokuwiki](https://www.dokuwiki.org/)、[Wiki.js](https://js.wiki)、[Notion](https://www.notion.so) 等等。

&emsp;&emsp;其中，Dokuwiki 是一个基于 PHP 的、可以百科站点、团队站点两用的开源程序。笔者比较喜欢 Dokuwiki 的一点是，**完全不需要数据库可以独立部署以及支持版本迭代**。这一点与“一切皆文件”的思想似乎很接近。（😂 实际上是因为笔者不愿意管理数据库） Wiki.js 虽然是基于 NodeJS 编写的，但还是需要连接数据库。Notion 是一款更像个人笔记的软件，无须自行部署，只需要使用 Web 端或者客户端编辑即可。Notion 本身支持很多开放性的功能，甚至说还能和 [zotero](https://www.zotero.org) 一起管理文献。如果非常熟悉 Notion 的各种功能，可能用 Notion 来搭建一个百科也非常合适。不过，笔者觉得要达到这种程度可能需要像学习宇宙无敌的 IDE -- Visual Studio 一样复杂。

&emsp;&emsp;Dokuwiki 的设计有点类似于 VS Code 的设计哲学，本体提供的只是最基本的、最简单的功能。如果你想要其他功能或者改变样式，你可以通过安装插件或者主题的方式来实现。而 Dokuwiki
官方和社区作者提供了比较丰富的插件和主题，能够有很好的可扩展性和 DIY 可能。就比如本文准备要为 Dokuwiki 接入的 LDAP 认证，实际上也是官方提供的插件之一。默认程序是已安装 LDAP Auth Plugin 插件的，我们只需要再做一些简单设置即可接入 LDAP 认证。

## 实践

&emsp;&emsp;为了更加简便地实现 Dokuwiki 接入 LDAP 认证，这里采用了预先准备好的 Docker 镜像 -- [shuosc/dokuwiki](https://hub.docker.com/r/shuosc/dokuwiki)。如果感兴趣的话，可以访问笔者维护的 [shuosc/docker-dokuwiki](https://github.com/shuosc/docker-dokuwiki) 项目了解更多关于该镜像的构建细节。

### 预先准备的环境

- Docker 环境
- docker-compose 工具

### Dokuwiki

#### 创建实例

&emsp;&emsp;使用下面的 docker-compose.yml 文件和 `docker-compose up -d` 命令启动一个 Dokuwiki 实例。这里的端口映射可以根据喜好或实际情况自行调整。

```yaml
# docker-compose.yml

version: '2'
services:
  dokuwiki:
    image: shuosc/dokuwiki:latest
    ports:
      - 80:80
    environment:
      - DIR=wiki
    volumes:
      - ./data:/opt/data
```

#### 运行验证

&emsp;&emsp;访问 [http://localhost/wiki/](http://localhost/wiki/) 即可看到如下图所示的 Dokuwiki 实例首页。这里其实是已经使用默认的 admin 用户登录之后的页面了。

> info "小提示"
> &emsp;&emsp;shuosc/dokuwiki 默认管理员 admin 的初始密码为 admin。如果容器实例可被外部网络访问，出于安全性考虑建议在运行后及时修改成强密码。

![首页 Home Page](https://i.luish.cc/blog/nNCoC5.webp)

### 配置 LDAP 登录

#### 安装 LDAP 支持

&emsp;&emsp;由于 [shuosc/dokuwiki](https://hub.docker.com/r/shuosc/dokuwiki) 镜像本来不是为 LDAP 认证构建的，没有安装 LDAP 认证所需的 `php7-ldap` 库，所以需要在启动实例后进入容器内部安装一下，并重启容器实例生效。

```shell
# 进入容器实例
docker exec -ti <id> /bin/bash 

# 默认用户为 root
apt update && apt install -y php7-ldap

# 退出容器实例后执行
docker restart <id>
```

> info "小提示"
> &emsp;&emsp;由于 shuosc/dokuwiki 镜像默认采用的 USTC（中科大）的软件源，笔者在安装 php7-ldap 库时遇到 Not Found 的错误。如果你也遇到了，可以将容器实例的软件源切换到其他软件源，比如执行 `sed -i "s/ustc/nju/" /etc/apt/sources.list` 即可从 USTC 切换到 NJU（南京大学）软件源。  
> &emsp;&emsp;另外，这样的安装只是一种临时的解决方案，销毁并重建容器实例后仍然没有 php7-ldap 库。因此后续 shuosc/dokuwki 镜像将会增加这一支持。

#### 设置 LDAP

&emsp;&emsp;在登录成功后，可以如上步中图中所示点击右上角**管理**按钮进入**管理页面**。

![管理界面 Admin Page](https://i.luish.cc/blog/ABVW2W.webp)

&emsp;&emsp;这里可以先点击**扩展管理器**确认一下 **LDAP Auth Plugin** 插件是否已预安装。这里由于是启用后的截图，所以右边没有**卸载**和**关闭**按钮以及启用的提示。

![扩展管理器 Plugins Manage Page](https://i.luish.cc/blog/ysQEBr.webp)

&emsp;&emsp;返回刚才的**管理页面**，点击**配置设置**按钮即可进入完整的配置设置。如下图所示是 LDAP 认证部分的配置，在实际页面的比较靠后的位置可以找到。

![LDAP 设置 LDAP Settings](https://i.luish.cc/blog/hnMxkY.webp)

&emsp;&emsp;上图是默认的配置，我们需要填一下其中的一些条目，内容如下（其他保持默认即可）：

| 条目 | 内容 |
| --- | --- |
| server | ldap.example.com |
| usertree | ou=People,dc=example,dc=com |
| userfilter | (&(objectClass=posixAccount)(uid=%{user})) |
| version | 3 |
| binddn | cn=admin,dc=example,dc=com |
| bindpw | xxxxxxxxxxx |
| modPass | unchecked |

#### 启用 LDAP

&emsp;&emsp;上步填写的是 LDAP 相关配置信息，这里还需要将默认的**认证方式**（authtype）从 authbasic 切换到 authldap，如下图所示。另外，为了保证 LDAP 的管理员用户可以访问 Dokuwiki 的管理页面，这里也需要指定**超级用户**（superuser）的用户名。

> info "小提示"
> &emsp;&emsp;由于启用了 LDAP 认证，这里应该要如下图所示停用 Dokuwiki 的**注册**功能，

![启用 LDAP Enable LDAP](https://i.luish.cc/blog/zW5Fgq.webp)

### 其他

&emsp;&emsp;为 Dokuwiki 接入 LDAP 认证之后，所有指定组目录下的所有 LDAP 用户都可以正常登录 Dokuwiki。但由于 LDAP 只提供了用户登录验证，Dokuwiki 相应页面的权限仍然需要使用 Dokuwiki 自带的**用户管理器**来管理。具体操作可以 [Dokuwiki Manual](https://www.dokuwiki.org/start?id=zh:manual) 来了解更多。

## 参考资料

- [LDAP Auth Backend: OpenLDAP Examples](https://www.dokuwiki.org/auth:ldap_openldap)
- [Dokuwiki with LDAP error: User authentication is temporarily unavailable](https://stackoverflow.com/questions/24322779/dokuwiki-with-ldap-error-user-authentication-is-temporarily-unavailable)