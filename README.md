# Spam_detection

## 电子邮件协议

1. SMTP

2. POP3

3. IMAP

   存在的缺陷：

## 发件人伪造

1. ### 原理

&emsp;&emsp;`SMTP`不支持邮件加密、完整性校验和验证发件人身份。由于这些缺陷, 发送方电子邮件信息可能会被网络传输中的监听者截取流量读取消息内容导致隐私泄漏, 也可能遭受中间人攻击(`Man-in-the-Middle attack`, `MitM`)导致邮件消息篡改, 带来网络钓鱼攻击. 

&emsp;&emsp;为了解决这些安全问题应对日益复杂的网络环境, 邮件社区开发了诸多电子邮件的扩展协议, 例如`STARTTLS`, `S/MIME`, `SPF`, `DKIM`和`DMARC`等协议。 当前的邮件服务厂商大都也是采用以上扩展协议的一种或几种组合, 辅以应用防火墙、贝叶斯垃圾邮件过滤器等技术, 来弥补电子邮件存在的安全缺陷.

&emsp;&emsp;你可以理解为 `SMTP` 本身有毛病，机密性和安全性都不够，所以就多了几个插件来帮他:

> 1. 提高传输的机密性，包括端对端加密 –> `STARTTLS`, `S/MIME`
>
> 2. 邮件的身份验证 –> `SPF`, `DKIM`, `DMARC`

&emsp;&emsp;电子邮件欺诈本质上就是邮件接收端没法判断这一封邮件来源是否合法。所以这边就涉及到了邮件的身份验证这一块。而且虽然`STARTTLS`和`SMTP-STS`保证了邮件在传输过程中的加密, 防止遭受窃听读取, 但是其仍无法解决发件方身份伪造、消息篡改等问题。 所以邮件的身份验证这一块尤为重要。

2. ### SPF

&emsp;&emsp;SPF，全称为 Sender Policy Framework，即发件人策略框架。根据 **SMTP 的规则，发件人的邮箱地址是可以由发信方任意声明的**,这显然是极不安全的。

**SPF 出现的目的，就是为了防止随意伪造发件人**。

**原理：**

> &emsp;&emsp;SPF 记录实际上是**服务器的一个 DNS 记录**，使用电子邮件的头部信息中的 **'Return Path' 或 'Mail From'** 这两个邮件头里的**域名**来结合**真正提供这个邮件的服务商 DNS 里面的记录去验证发送邮件服务器是否是冒充行为**。原理其实很简单：
>
> &emsp;&emsp;假设邮件服务器收到了一封邮件，来自主机的 IP 是`173.194.72.103`，并且声称发件人为`email@example.com`。为了确认发件人不是伪造的，邮件服务器会去查询`example.com`的 SPF 记录。如果该域的 SPF 记录设置允许 IP 为`173.194.72.103`的主机发送邮件，则服务器就认为这封邮件是合法的；如果不允许，则通常会退信，或将其标记为垃圾/仿冒邮件。
>
> &emsp;&emsp;因为不怀好心的人虽然可以「声称」他的邮件来自`example.com`，但是他却无权操作`example.com`的 DNS 记录；同时他也无法伪造自己的 IP 地址。因此 SPF 是很有效的，当前基本上所有的邮件服务提供商（例如 Gmail、QQ 邮箱等）都会验证它。

**记录查询：**

&emsp;&emsp;spf记录本质上是服务器上的TXT类型的DNS记录，因此要查询SPF记录只需要使用dig命令进行查询：

~~~python
>> dig -t txt 163.com		# 查询163邮箱的spf
ANSWER SECTION:
163.com.		1160	IN	TXT	"v=spf1 include:spf.163.com -all"  # 其中的一条记录,代表了spf规则遵从spf.163.com的，在其中没有包含的都不允许接收
~~~

&emsp;&emsp;以查询xxx@163.com的spf记录为例，完整的查询过程如下：

~~~python
>> dig -t txt 163.com		# 查询163邮箱的spf
ANSWER SECTION:
163.com.		1160	IN	TXT	"v=spf1 include:spf.163.com -all"  # 其中的一条记录,代表了spf规则遵从spf.163.com的，在其中没有包含的都不允许接收
>> dig -t txt spf.163.com 
ANSWER SECTION:
spf.163.com.		7283	IN	TXT	"v=spf1 include:a.spf.163.com include:b.spf.163.com include:c.spf.163.com include:d.spf.163.com include:e.spf.163.com -all"   # spf规则为a、b、c、d、e几个域名spf规则的和
>> dig -t txt a.spf.163.com   # 这里以a.spf.163.com为例，实际上还应对其他4个域名的spf记录进行访问
ANSWER SECTION:
a.spf.163.com.		60	IN	TXT	"v=spf1 ip4:220.181.12.0/22 ip4:220.181.31.0/24 ip4:123.125.50.0/24 ip4:220.181.72.0/24 ip4:123.58.178.0/24 ip4:123.58.177.0/24 ip4:113.108.225.0/24 ip4:218.107.63.0/24 ip4:123.58.189.128/25 ip4:123.126.96.0/24 ip4:123.126.97.0/24 -all"     # 明确了可以接收的ip地址
~~~

**生效时间：**

> SPF 记录本质上是一个 DNS 记录，所以并不是修改之后立即生效的——通常需要几个小时的时间。



3. ### DKIM

&emsp;**&emsp;DKIM 域名密钥识别邮件标准(`Domain Keys Identified Mail`)**是一种通过**检查来自某域签名的邮件标头来判断消息是否存在欺骗或篡改**的检测电子邮件欺诈技术。

**原理：**

&emsp;&emsp;利用加密签名和验证的原理, 在发件人发送邮件时候, 将与域名相关的私钥加密的签名字段插入到消息头, 收件人收到邮件后通过DNS检索发件人的公钥(`DNS TXT记录`), 就可以对签名进行验证, 判断发件人地址及消息的真实性. 简单的来说，就是**使用 DKIM 发送的电子邮件包括一个 `DKIM-Signature` 标头字段**，该字段**包含邮件的加密签名表示形式 (里面有私钥)**。**接收邮件的提供商可以使用公有密钥（这在发件人的 DNS 记录中发现）对签名进行解码。然后，电子邮件提供商使用此信息来确定邮件是否是真实的。**

![image](https://raw.githubusercontent.com/AnchoretY/images/master/blog/image.re1j794ecso.png)

&emsp;&emsp;举个例子， 比如我们正常的 gmail 或ses 发送邮件时，通过和域关联，自动生成的私钥加密邮件，生成DKIM-Signature签名，然后将DKIM-Signature签名及相关信息插入到邮件标头，发送给接收方, 而接收方收邮件时，通过DNS 查询获得公钥，验证邮件DKIM签名的有效性，从而确认在邮件发送的过程中，防止邮件被篡改，保证邮件内容的完整性。所以只要能用找到的公钥，去解标头字段里面的私钥信息，就说明是真实的，其实就是非对称的端对端加密，只不过我们把公钥暴露在域名的 DNS TXT记录中，接收邮件的提供商可以在公网上查到。

**邮件中的DKIM-Signature格式介绍：**

&emsp;&emsp;DKIM的基本思想是发件人域中的邮件服务器在邮件中添加数字签名，收件人通过签名验证邮件是否由邮件服务器发送，签名大概格式如下：

~~~shell
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
      d=dhl.com; l=1850; s=20140901; t=1452769712;
      h=date:from:to:message-id:subject:mime-version;
      bh=yCbsFBJJ9k2VYBxKGgyNILalBP3Yzn1N8cMPQr92+zw=;
      b=bnuXrH/dSnyDR/kciZauK4HTgbcDbSFzmHR78gq+8Cdm20G56Ix169SA...
~~~

&emsp;&emsp;在邮件签名中，最重要的部分是：

~~~
1. d是这一个域的签名。这一部分是在DMARC检查签名中的域是否和发送者的域相匹配中用到。
2. h是应该包含在签名中的邮件头的字段列表
3. bh是邮件正文的hash
4. l是hash在邮件正文中所占的字节数。这一选项是可选的。
5. b是签名本身，包括'h'给出的字段，以及DKIM-Signature头本身。由于头部保保存的是整个正文的hash，所以b这一参数也可以作为电子签名。
~~~

&emsp;&emsp;除此之外，签名中还包含了使用的加密算法’a’，以及用于在DNS中寻找RSA密钥的参数’s’，规范性的方法’c’，以及时间戳’t’.

**缺陷：**

> 1. DKIM 只能验证发件人地址来源的真伪, 无法辨识邮件内容的真实性. 
> 2. 如果接收到无效或缺少加密签名的消息, DKIM 无法指定收件人采取什么措施. 
> 3. 私钥签名是对本域所有外发邮件进行普遍的签名, 是邮件中继服务器对邮件进行 DKIM 签名而不是真正的邮件发送人, 这意味着, DKIM 并不提供真正的端对端的电子签名认证。

**DKIM设置：**

&emsp;&emsp;设置其实就是在 DNS TXT 设置公钥

![image](https://raw.githubusercontent.com/AnchoretY/images/master/blog/image.0nq4xcb1omdq.png)

&emsp;&emsp;其中 p 字段后面就是一大段的公钥。 

**生效时间：**

> DKIM本质上也是一种TXT DNS记录，因此修改后并不是立即生效的，而是要过几个小时后生效。

**DKIM绕过：**

1. 欺骗邮件头部

   利用邮箱客户端和DKIM显示与加密头部签名内容的策略不同进行欺骗邮件头部。

   > 原理：当存在多个Subject、Content-Type时，Gmail等邮箱客户端都只会显示第一个，而DKIM仅仅会使用最后一个。

   > 如果标题中包含了两个’to’字段，并且两者都应该写被保护，那么就应该在签名的’h’选项中包含’to’两次。

   例如下面的邮件：

   ~~~
   DKIM-Signature: v=1; h=from:to:cc:subject:content-type; ...
       From: <dkim-test@chksum.de>
       To: knurrt.hase@gmail.com
       Subject: 20170920:1755 - good
       Content-type: multipart/mixed; boundary=foo
       Date: Wed, 20 Sep 2017 17:55:18 +0200
   
       --foo
       Content-type: text/plain
   
       some text
       --foo--
   ~~~

   使用安装DKIM插件的邮件客户端Thunderbird查看邮件时显示如下：

   ![image](https://raw.githubusercontent.com/AnchoretY/images/master/blog/image.i6bk81v98dr.png)

   但是通过添加额外的Subject以及Content-Type头部，它显示的结果会发生不同：

   ~~~
   Subject: Urgent Update at http://foo
       Content-type: multipart/mixed; boundary=bar
       DKIM-Signature: v=1; h=from:to:cc:subject:content-type; ...
       From: <dkim-test@chksum.de>
       To: knurrt.hase@gmail.com
       Subject: 20170920:1755 - good
       Content-type: multipart/mixed; boundary=foo
       Date: Wed, 20 Sep 2017 17:55:18 +0200
   
       --foo
       Content-type: text/plain
   
       some text
       --foo--
   ~~~

   我们可以观察到subject是不同的，并且邮件正文已经没有了，但是DKIM验证结果还是正确的。

2. 欺骗邮件正文（任意控制文件内容）

   &emsp;&emsp;如果邮件发送法在签名中使用了**l选项**，那么这种攻击就可以实现了，‘I’属性表示正文hash所占的字节数。在DKIM中很多情况下搜只保保证了一些选项被签名保护，但是签名并没有防止添加另外的字段，所以字原始邮件正文中添加任何东西都不会使签名失效。

   &emsp;&emsp;我们可以在邮件中继续添加另外一个Date、To、Message-Id等等其他选项，并且可以更改当前的Content-Type、在正文中添加任意数据。这样都不会使签名无效。下面的例子中更改的内容为红色，原始邮件为黑色和蓝色：

   ![image](https://raw.githubusercontent.com/AnchoretY/images/master/blog/image.0a13630dzand.png)

   &emsp;&emsp;在”signed-by”字段中可以看到签名仍然有效。而且如果我们查看邮件的来源，Gmail会提供一个很好的摘要，其中包括攻击者设置的Date以及Messahe-Id.由于DKIM签名与显示的发件人DMARC的域相匹配，所以即使邮件未通过DHL的邮件服务器发送，也会成功：

   ![image](https://raw.githubusercontent.com/AnchoretY/images/master/blog/image.yrmo1sa2t3r.png)

   > 这一问题是非常严重的一个邮件安全问题，要想进行避免这个问题，需要发送方与接收方共同努力：
   >
   > - 发送方：签名本身需要包含可能会影响邮件显示的所有邮件头
   > - 接收方：对于没有包含在签名中的标题应该格外小心

   

【参考文献】

- [绕过DKIM验证，伪造钓鱼邮件- 知乎](https://zhuanlan.zhihu.com/p/29965126)
- [电子邮件欺诈防护之SPF+DKIM+DMARC | Zach Ke's Notes](https://kebingzao.com/2020/03/17/mail-spf-dkim-dmarc/)

