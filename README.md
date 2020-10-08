# Spam_detection



### 发件人伪造

1. #### 原理



发件人伪造的检测方式：

2. #### SPF

&emsp;&emsp;SPF，全称为 Sender Policy Framework，即发件人策略框架。根据 **SMTP 的规则，发件人的邮箱地址是可以由发信方任意声明的**,这显然是极不安全的。

**SPF 出现的目的，就是为了防止随意伪造发件人**。

SPF 的原理是使用电子邮件的头部信息中的 'Return Path' 或 'Mail From' 这两个邮件头里的域名来结合真正提供这个邮件的服务商 DNS 里面的记录去验证发送邮件服务器是否是冒充行为。

**原理：**

> SPF 记录实际上是服务器的一个 DNS 记录，原理其实很简单：
>
> ​	假设邮件服务器收到了一封邮件，来自主机的 IP 是`173.194.72.103`，并且声称发件人为`email@example.com`。为了确认发件人不是伪造的，邮件服务器会去查询`example.com`的 SPF 记录。如果该域的 SPF 记录设置允许 IP 为`173.194.72.103`的主机发送邮件，则服务器就认为这封邮件是合法的；如果不允许，则通常会退信，或将其标记为垃圾/仿冒邮件。
>
> ​	因为不怀好心的人虽然可以「声称」他的邮件来自`example.com`，但是他却无权操作`example.com`的 DNS 记录；同时他也无法伪造自己的 IP 地址。因此 SPF 是很有效的，当前基本上所有的邮件服务提供商（例如 Gmail、QQ 邮箱等）都会验证它。

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