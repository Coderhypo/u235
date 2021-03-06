---
title: "密码安全:存储/传输/安全问题设计"
date: 2018-1-2T22:35:30+08:00
tags: "安全"
---

 # 存储设计
设计原则: **密码存储必须杜绝获取明文之可能**

用户的密码通常具有一定规律,一般用户会对若干不同账户使用相同密码,不论以何种方式导致明文泄漏,就意味着用户的其他一系列账户也受到同样威胁. 在一些国家,对于用户密码存储有着相关规定.

### 1. 密码不以明文存储

### 2. 密码不以简单hash方式存储

  hash方式包括hash算法选型、混淆手段及迭代次数。

#### 算法选型

AES256是较安全的算法, MD5和SHA128属于较简单的算法, 一方面是因为它们的安全空间相对于如今的计算能力而言,  应对生日攻击(Birthdate Attack)的安全性已经变弱. 另一方面, 它们存在的年代已久, 已经有了较大规模的hash数据库, 可能通过简单的数据库查询就可以找到对应明文

[生日攻击:](https://zh.wikipedia.org/wiki/%E7%94%9F%E6%97%A5%E5%95%8F%E9%A1%8C) 如果一个房间裡有23个或23个以上的人，那么至少有两个人的生日相同的概率要大于50%
以MD5为例,理论上讲它存在2^64中可能性,但其实只需尝试2^32种可能就有很大可能找到碰撞,而借助一些方法还可以将尝试次数降低到2^16以内

### 混淆手段

直接对明文,或者是简单加上固定的一段字符串进行hash运算并存储是不明智的选择, 这带来两个问题:一方面, 攻击者获取到hash值后, 可以自行破解/查询数据库直接获取到明文, 另一方面, 如果你的数据库中有两个用户碰巧使用相同的密码,那么攻击者就会意识到这可能是一个弱密码(如123456). 获取到数据库的攻击者就很容易找到它,并进行集中破解.
比较可靠的办法是对每个用户使用一段随机的字符串(通常称为salt),在运算hash时将其与明文连接运算.

### 迭代次数

如果只进行一次hash,以MD5为例,它存在2^64中可能,某个数据库中可能就存在着它的明文. 如果使用两次,那么就进一步缩小了攻击者直接通过hash找到明文的可能性. 当然,这并不能阻止攻击者通过字典攻击找出诸如"abcd1234"这样的简单密码.

### 密码传输

密码传输存在于两个阶段: 注册和认证. 

#### 1. 注册

Django和很多网站在注册期间密码都是通过明文传输的, 某些网站甚至没有使用HTTPS保护这些密码明文. 

防范措施:
* 由服务器发出一段salt, 客户端将其与密码明文运算后发送给服务器.
* 通过HTTPS保护注册过程

服务端存储 K = hash( password, reg_salt )

#### 2. 认证

不论服务器上以和何种方式存储了用户密码, 用户和服务器之间总要通过一种方式进行验证. 这中间存在几种可能的攻击:

* **重放攻击**: 攻击者监听用户发出的内容, 并将其用于自己的认证.  
   防范方式:由服务器发送给用户一段随机数值(salt), 由用户运算后在服务端进行验证. 由于这个值每次都不同,攻击者无法将其用于自己的身份认证. 这种方式也被称为challenge
* **字典/穷举攻击**: 攻击者监听整个通讯过程,得知了此次的salt和用户发送的hash值,自行进行穷举/字典查询. 对于某些简单密码, 例如纯数字密码, 这种穷举只需要极少运算即可. 例如在我上大学期间, 学校网络供应商提供的是一种6位数字密码的上网卡, 其密码验证方式就是通过此种方式进行, 只需要根据salt,穷举6位数字(10^6, 较差的电脑也只需要数秒), 即可获得密码明文.
防范方式: 通过HTTPS进行通讯加密,防止嗅探到明文.

* **伪造服务器攻击**: 攻击者谎称自己是服务器,并向用户发送一段有缺陷的salt. 假如混淆方法是字符串连接,那么攻击者指定salt为空字符串, 就获得了明文的直接hash值, 大大提高了攻击成功几率.
防范方式:通过HTTPS进行对服务端的身份认证，使用字符串补齐等更复杂的方式

认证过程:
1. 服务端告知客户端注册时的盐值及本次盐值: reg_salt, challenge_salt , 并运算:
    server_k = hash( K, challenge_salt )
2. 客户端进行运算,并发送给服务端
   K = hash( password, reg_salt )
   client_k = hash( K, challenge_salt )
3. 服务端比较 client_k与server_k


# 重置密码安全问题设计

现在比较流行的方法是使用手机或安全邮箱进行密码重置. 某些还会提供一种通过安全问题进行重置.

考虑到密码的两种主要泄漏方式:偷看(shoulder surfing)和社会工程(social engineering), 查户口式的"安全问题"并不安全. 例如这些问题:

* 你妈姓什么?
* 你毕业于哪个大学?
* 你喜欢的歌手叫什么名字?
* 你喜欢的运动是什么?

在某些情况下, 这些所谓的安全问题只需要简单的调查就可以得到答案. 有趣的是, 早年的网易邮箱就曾经使用过最后一个问题"你喜欢的运动是什么", 然后我轻而易举的差点重置了临班某同学的邮箱密码

安全问题设计上应当注意以下方面:
* 能够通过某些文档获取的资料不应当用于安全问题
  例如很多公司面试期间会要求填写教育史,父母信息等, 这些问题不应当用于安全问题
* 用户身边的人能够通过日常交往得到答案的, 不应成为安全问题
  例如在工作场合, 用户喜欢的食物/运动, 以及用户的家乡等,都是很容易打听到的. 相对而言,初恋的姓氏就不是一个容易泄漏的话题
* 不应当设计某些隐私问题
  一旦泄漏可能对用户造成困扰. 较妥的办法是以hash形式存储这些问题的答案,并且问问题时不应当要求用户回答"初恋的名字"这种问题, 更好的问题是"初恋的姓氏"或者"初恋的姓氏首字母"
* 不使用固定的问题. 例如可以存储6个问题的答案,每次随机抽取三个, 增加攻击者的打探难度.