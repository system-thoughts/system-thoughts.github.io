---
title: Head first for mutt
date: 2021-03-31T19:21:52+08:00
tags: [mutt, 社区]
categories: tools
---

给社区发送邮件讨论补丁时，选择一个趁手的客户端会极大提高沟通效率。曾经我通过"git send-email + thunderbird"的工具组合进行邮件的发送和读取，[thunderbird如同outlook的clone版本](https://www.kernel.org/doc/html/v4.10/process/email-clients.html)，对于使用GUI客户端的用户极为友好。然而大多数内核开发者还是会选用mutt邮件客户端，其可配置性极佳，完全命令行的交互方式对于kernel hacker来说是极为友好的。

<!-- more -->

## what is mutt and how dose it work?
> "All mail clients suck. This one just sucks less." -- Michael Elkins, circa 1995

mutt是一个邮件客户端，直接与终端用户进行交互，Internet规范和RFC使用术语MUA(Message User Agent)描述邮件客户端。MUA可以是桌面应用程序，如Microsoft Outlook、Thunderbird等，也可以是基于Web的应用程序，如Gmail、Hotmail等（后者也称为Webmail）。用户使用mutt应该能够完成基本的邮件编写、接收、发送。但mutt将这些基本功能都模块化并交给其他程序完成，如mutt可以配置环境中的编辑器作为撰写邮件的工具，比如我会配置趁手的vim。为了确定还需要哪些辅助程序帮助mutt完成基本的邮件收发功能。我们需要了解email的传送过程：为了将邮件从发送者通过Internet传递给接收者，还需要MTA(Message Transfer Agent)进行邮件转发，一个简略的邮件传送流程：
```
sender -> transfer -> recipient
MUA → MTA → … → MTA → MUA
```
可以将上述流程进行细化：
```
push steps: →
pull steps: →→
MUA → MSA → MTA → … → MTA → MDA →→ MRA →→ MUA
```
这里对上述术语做一个清晰的解释：
* MUA(Message User Agent): MUA是终端用户管理和读取存储到用户邮箱的电子邮件的前端，并可在其中进行邮件编辑(compose)且通过MSA发送出去。
* MTA(Message Transfer Agent)：MTA对邮件进行排队(queue)、接收(receive from)、并发送(send)给下一MTA。这包括邮件路由(routing mail)、排队(queuing)和重试(retrying)，如果下一MTA未能立即接收或者处理email或者便向原始发件人发送通知。
* MSA(Message Submission Agent)：接收来自MUA发送的邮件，作为SMTP client，仅能发送email，而不能像Full-fledged MTA接收email进行转发。MTA使用端口25，MSA不仅可以使用端口25，还有offical port 587。MSA使用ESMTP(SMTP协议的变种)。MSA有时候也形象地称为Mail Sending Agent。
* MDA(Message Delivery Agent)：MDA收到邮件之后不会往下转发，即邮件已经投递到位。MDA将邮件放入收件箱(incoming mailbox)，MDA可以调用`procmail`
* MRA(Mail Retrieval Agent)：与远程邮箱建立连接并获取邮件到本地使用，MRA是从邮件接收者的视角来命名，IETF不支持该术语，认为其是MUA[1]。

若是要通过mutt完成邮件的收发，还需要具备MSA和MRA功能的软件配合mutt。显然，在我们工作的客户端不需要Full-fledeged MTA完成邮件的中转，我们仅需将邮件发出即可。另外，我们也不需要MDA对邮件进行投递（如果您是为个人或者公司搭建邮件服务器，就当我没说🙊），因为我们使用的邮箱，诸如Gmail的邮件服务器已经完成邮件投递工作，我们仅需通过POP3/IMAP协议接收邮件。然而，我们通过搜索引擎配置搜索mutt的配置，经常会出现各种`mutt + x + y + ...`的搜索结果令人眼花缭乱。这是因为同一功能，你的选择很多，就拿MSA来说，您可以选择`postfix`、`sendmail`、`msmtp`等[2]。而且，同一软件具备的多种功能，如`sendmail`、`postfix`不仅具有MTA功能，还集成了MDA功能。😤害，这不是违背了Unix哲学"one task per tool"！是的，甚至连mutt也背离初心:
> mutt was developed with the concept of "one task per tool", enabling performance through combination with other high quality modular programs. 

但是mutt现在实际变成了当初它讨厌的样子😅：
> ... or why mutt should not but slowly becomes an "all-in-1" program.

mutt已经引入了最简单的MRA、MSA功能支持SMTP、POP3、IMAP进行邮件收发。为何会背离初心：
> "This entire concept is rubbish and the mutt developers are just plain lazy [to add all the good stuff]."

好吧，看来群众的力量是很难承受的……
另外有些功能的界限实际难以区分，比如MDA和MRA。

OK, 说了这么多关于邮件的基本概念，let's get hands dirty，第一步先安装mutt（本文使用CentOS8环境，Ubuntu用户应该会更方便）:
```bash
yum install mutt
```

## Basic Configuration
mutt默认从系统范围的配置文件（“/etc/Muttrc”）读取其配置的默认值，该文件通常控制系统设置并为所有用户提供可行的默认配置。然后读取个人配置文件（“~/.muttrc”或“~/.mutt/muttrc”），这样，个人设置会根据需要覆盖系统设置。我们采用`~/.mutt/muttrc`对mutt进行配置。

mutt中的变量通过`set var=value`方式配置。存在仅需设置yes/no的`toggle var`（布尔变量），你可以为`toggle var`设置"ask-yes"或者"ask-no"以指定每次使用时给定默认答案进行提示。也可以通过 "unset"、"set var"、"set novar"简化布尔变量的设置。需要注意的是，你配置的mutt变量对应的功能在编译mutt时已启用才会成功设置，否则你会得到“unknown variable”的告警。所有的mutt配置参数分类可参考[ConfigList](https://gitlab.com/muttmua/mutt/-/wikis/VarNames/List)。
下面给出最基本的mutt配置：
```
# Basic configuration
set ssl_force_tls = yes
set abort_nosubject = no
set timeout = 10
set mail_check = 60
set sort = "reverse-date-received"
```

`ssl_force_tls`表示mutt与所有的远端服务器的连接需要通过TLS协议加密，如果不能成功建立连接，中断通信。`mutt -v | grep tls`能够打印`--with-gnutls`表示mutt支持TLS。

`abort_nosubject`的默认值是"ask-yes"，表示我们发送邮件还未写邮件主题时，该配置将会提示，默认值为"yes"。将该项设置为"no"，便无需确认。

mutt每次接收到键盘输入时便会更新所有目录的状态。我们也希望即使在空闲时也能收到新邮件的通知，而不需要按下按键。控制这种行为的变量是`timeout`。它表示等待用户输入的最大时间，以秒为单位。如果在指定的时间内没有接收到用户输入，便执行更新操作。变量的默认值是600秒，表示在没有输入的情况下，每10分钟接收一次更新。默认值太高，我们设置为10。

如前所述，每次收到用户输入时，mutt都会查找更新。键盘活动太频繁会导致过多的访问操作，为了限制这个频率，使用`mail check`变量。该变量表示两次扫描之间的最小时间(以秒为单位)。该变量的默认值是5，即使经常按下键，mutt也将每5秒搜索一次新邮件。该值还是太小，尤其是在多邮箱的场景下，可能因为频繁访问降低速度。

默认情况下，索引菜单(显示消息列表)中的电子邮件按日期升序排序，因此更新的电子邮件将显示在底部。要更改电子邮件的排序方式，我们可以使用和设置`sort`变量的值。在本例中，我们设置`reverse-data-received`让更新的电子邮件出现在列表的顶部。其他参数也可以用作排序因子，例如subject和size。


## Retrieve email
MUA使用POP3/IMAP协议从邮件服务器下载邮件。两者的区别如下[3]：
* POP3(Post Office Protocol 3)：仅仅下载邮件服务器的inbox目录中的邮件到本地，不会下载sent、draft、deleted目录下的邮件。并且POP3不会同步，并且当email被下载到一台设备上时，email会从邮件服务器中删除。
* IMAP(Internet Message Access Protocol)：允许在多个客户端上查看邮件（同步），IMAP会将邮件缓存到本地。IMAP还会同步各个设备的目录结构

![](POP3.PNG)
上图展示了两台电脑都从同一email账号下载邮件（均使用POP3协议），两台电脑上的目录结构截然不同，因为POP3不会同步两台电脑中的目录。当有新邮件到来时，第一台电脑先下载了邮件，邮件便会从邮件服务器中删除，第二胎电脑是获取不到新邮件的。不过后面这个问题还好，很多邮件客户端可以设置`leave a copy of messages on the server`。

POP3和IMAP的比较，如下表所示：

| metrics |  POP3  | IMAP |
| -- | -- | -- |
| view without Internet | Yes | NO |
| All the folders can be seen | NO | YES |
| All the folders and emails are synchronized | NO | YES |
| Email stored on the mail server | NO | YES |

由于IMAP默认在本地仅缓存(cache)邮件，并不下载邮件，所以默认在无网络的条件下，使用IMAP的MUA是无法阅读邮件的，但是现在很多使用IMAP的MUA都可以设置将邮件下载到本地，而非仅仅缓存。
POP3协议由于会将邮件下载到本地（下载会删除邮件服务器中的邮件），从而能够节约邮件服务器的空间，然而本地下载的邮件需要备份以免磁盘损坏邮件丢失。

`mutt -v | grep IMAP`能够打印`+USE_IMAP`表示mutt支持IMAP。当前mutt版本已经支持IMAP。那么，我们直接使用mutt自带的IMAP功能下载邮件，关于IMAP相关的配置如下：
```
set from = "foo.bar@gmail.com"
set realname = "Foo Bar"

# Imap settings
set imap_user = "foo.bar@gmail.com"
set imap_pass = "<mutt-app-specific-password>"

# Remote gmail folders
set folder = "imaps://imap.gmail.com/"
set spoolfile = "+INBOX"
set postponed = "+[Gmail]/Drafts"
set record = "+[Gmail]/Sent Mail"
set trash = "+[Gmail]/Trash"
```
`from`指定的是[email header](https://whatismyipaddress.com/email-header)，此处填写你的邮箱地址。`realname`填写你的真实姓名。

[`imap_user`](http://www.mutt.org/doc/manual/#imap-user)是在IMAP服务器上访问其邮件的用户名，和`from`变量保持一致，此处以gmail账户为例。[`imap_pass`](http://www.mutt.org/doc/manual/#imap-pass)是IMAP账户的密码，谷歌要求不使用Oauth2身份验证方法的应用必须使用特定于应用程序的密码。为了能够从mutt访问我们的gmail帐户，我们必须[打开2-Step Verification](https://support.google.com/accounts/answer/185839)，随后[生成特定于应用程序的密码](https://support.google.com/accounts/answer/185833?hl=en)。

* [`folder`](http://www.mutt.org/doc/manual/#folder)：mailbox的默认位置
* [`spoolfile`](http://www.mutt.org/doc/manual/#spoolfile):新邮件到达mailbox时，归档的目录
* [`postponed`](http://www.mutt.org/doc/manual/#postponed)：存储待发送的邮件(草稿)的文件夹
* [`record`](http://www.mutt.org/doc/manual/#record)：存储已发送的邮件的文件夹
* [`trash`](http://www.mutt.org/doc/manual/#trash)：存储已删除邮件的文件夹

在客户端执行`mutt`打开邮箱查看gmail邮件：
![](收件箱.PNG)

## Send email
MTA进行邮件转发使用SMTP协议(Simple Mail Transfer Protocol)。如下这段话，我认为是对SMTP协议的功能的一个良好概括[4]。
> SMTP is basically a set of commands that authenticates and directs the transfer of email

更简单方式的记住S(ending) M(ail) T(o) P(eople)。下图展示了通过SMTP协议将邮件从SMTP client途径SMTP server传递到收件人的SMTP server，最终收件用户登录信箱通过POP3/IMAP下载邮件。
![](smtp.PNG)

MTA可以分为两类：仅转发(relay-only)、全功能(full-fledged)[5]。
* Relay-only MTAs: 或者称为Send-only MTAs、small MTAs(SMTP clients)[6]，此类MTA仅将你的email转发到另一个服务器，如果你像我一样仅仅想发送自己的Gmail邮件，此类MTA是最好的选择。SMTP client仅执行某些特定的功能，而不像Full-fledged MTA一样运行完成的SMTP服务器，占用大量的资源开销。这样的MTA不会监听传入的消息，尽在需要发送邮件的时候运行。small MTA通常作为MSA即邮件发送的第一站。
* Full-fledged MTAs：也称为mail hub，能够处理Internet邮件传送的所有细节，通常这类MTA从Relay-Only MTA接收邮件再进行转发。邮件在Internet的中间MTA转发漫游时，这些MTA使用full-fledged MTA比较合适。

本文仅考虑MSA即Relay-only MTA，选取软件作为MSA应该考虑如下因素：
* MTA是否可以对邮件进行排队，以便在出现故障时稍后发送
* MTA可以取代MDA?如果可以，它将处理来自系统的所有邮件。
* MTA是否支持连接到ISP SMTP服务器的要求？这些要求可能包括特定的身份验证或TLS。

mutt早已支持ESMTP/SMTP，`mutt -v | grep SMTP`能够打印`+USE_SMTP`表示mutt支持SMTP。当前mutt版本已经支持SMTP。那么，我们直接使用mutt自带的SMTP功能发送邮件，此时，mutt自己可以作为MSA。如果您需要选择功能更为复杂的MSA，请参考[SendmailAgents](https://gitlab.com/muttmua/mutt/-/wikis/SendmailAgents)。关于SMTP相关的配置如下：
```
# smtp
set smtp_url = "smtp://foo.bar@smtp.gmail.com:587"
set smtp_pass = $imap_pass
```
测试邮件发送:
```bash
[root@localhost ~]# proxychains4 echo "mail test" | mutt  -s "test email" buweilv@qq.com
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/local/lib/libproxychains4.so
[proxychains] DLL init
Could not connect to smtp.gmail.com (Connection refused).
Could not send the message.
```
邮件发送失败。使用`mutt -d`选项输出调试信息，调试级别从1~5，日志详细级别相应提高。调试信息输出到'~/.muttdebug*'。上面的发送失败很可能是因为网络不稳定。gmail smtp使用更为安全的TLS协议，端口为[587](https://support.google.com/mail/answer/7126229?visit_id=636857391298706033-411564613&rd=2#cantsignin&zippy=%2Csecurity-certificate-cn-error%2Cmy-email-client-is-crashing-or-emails-are-taking-too-long-to-download)。[ssl_ca_certificates_file](http://www.mutt.org/doc/manual/#ssl_ca_certificates_file)指定的文件包含了所有受信任的CA证书。使用这些CA证书之一签名的任何服务器证书也会被自动接受(仅GnuTLS)。使用`mutt -D`查看默认配置：
```bash
[root@localhost ~]# mutt -D | grep ssl_ca_certificates_file
ssl_ca_certificates_file="/etc/ssl/certs/ca-bundle.crt"
[root@localhost ~]# rpm -qf /etc/ssl/certs/ca-bundle.crt
ca-certificates-2020.2.41-80.0.el8_2.noarch
```
故使用TLS协议的SMTP服务应该安装`ca-certificates`。Ubuntu环境下的文件命名不同，包名也不同。

## Interact with mutt
之前我们使用简单的mutt命令行完成邮件的收发工作。但是，进入mutt交互界面之后，我们应该如何读取邮件，对邮件分类，并且在mutt客户端内编辑邮件、回复邮件呢？首先，我们得了解mutt界面的功能区域。
mutt提供不同的窗口(menu)与用户进行交互，这些窗口大多基于行(line-based)/条目(entry-based)或基于页面(page-based)。 基于行的窗口是所谓的“索引”(index)窗口（列出当前打开的文件夹的所有邮件）或“别名”(alias)窗口（允许您从列表中选择收件人）。 基于页面的窗口的例子是“pager”（一次显示一封邮件）或“帮助”窗口，其中列出了所有可用的绑定键。
下图以索引窗口为例，展示mutt交互界面的基本构成。图中的`context senstive`表示此处的内容和窗口类型有关。
![](mutt窗口.png)
交互界面主要由以下元素构成：
* context sensitive help line
* 窗口具体内容
* context sensitive status line
* 命令行：显示信息和错误消息以及提示和输入交互式命令

mutt共存在如下窗口(menu)：
* index: 启动Mutt时最先看到的窗口，展示当前打开邮箱中的email。index窗口是line-based，每一行从左到右分别表示邮件编号、标志（新电子邮件，重要电子邮件，已转发或回复的电子邮件，已标记电子邮件，...）、邮件发送日期、email大小、主题。如果mutt配置`set sort = "threads"`，邮件按照thread（对话）的形式层级展开：当您回复的电子邮件，被对方回复，您可以在下面的“子树”中看到对方的电子邮件。如果您订阅了邮件列表，这种显示方式非常有效。
* pager：负责显示电子邮件内容。pager的顶部有email header中的主要信息，如发件人，收件人，主题等。你可以配置mutt展示更多email header内容。email header下方便是邮件正文，如果邮件包含附件，会在邮件正文下方显示。如果附件就是文本文件，其内容会直接在pager中显示。为了有更好的观感，可以在mutt中配置在pager中为不同内容显示不同颜色。实际上，任何可以用正则表达式描述的内容都可以着色，例如网址、邮箱地址或表情符号。
* file browser：是本地或远程文件系统的接口，尚未遇到。
* sidebar：显示了所有邮箱的列表。该列表可以打开和关闭，它可以主题化，列表样式可以配置。
* help：旨在为用户提供快速帮助。它列出了键绑定的当前配置及其相关命令(commands)，包括简短的描述，以及当前未绑定函数(或者，可以通过Mutt命令提示符调用它们)。
* compose menu：撰写窗口具有一个拆分窗口，其中包含收件人、抄送人的信息。另外用户还可以对邮件进行加密、签名。
* attachment menu：mutt支持发送和接收任意MIME类型的消息，附件窗口详细地展示了邮件的结构。
* alias menu：帮助用户查找消息的收件人。对于需要联系很多人的用户来说，不需要完全记住地址或名字。mutt的别名机制以及别名窗口还具有按更短的别名(实际别名)对多个地址进行分组的功能，这样用户就不必手动选择每个收件人。

mutt作为文本邮件客户端，需要完全依赖键盘完成基本操作。当前介绍基于行/条目窗口的常规按键，以index窗口为例：
* 移动：k/j 上下移动, Z/z 上下翻页, =/* 跳转到第一封/最后一封邮件，<Number> 跳至序号处（不进入邮件）
* 基本操作：q退出当前窗口，<Enter>打开选中邮件，? 查看当前窗口的键绑定
* /在当前文件夹搜索

针对index窗口有些专有功能的按键：
* d:删除当前邮件, s:将邮件移动至指定文件夹, m:创建新邮件, r:回复当前邮件



## Reference
[1] Email agent (infrastructure) https://en.wikipedia.org/wiki/Email_agent_(infrastructure)#cite_note-modularmonolithic-schroder-1
[2] SendmailAgents https://gitlab.com/muttmua/mutt/-/wikis/SendmailAgents
[3] POP3 vs IMAP - What's the difference? https://www.youtube.com/watch?v=SBaARws0hy4&t=337s
[4] What is SMTP - Simple Mail Transfer Protocol https://www.youtube.com/watch?v=PJo5yOtu7o8
[5] Postfix vs. Sendmail vs. Exim https://blog.mailtrap.io/postfix-sendmail-exim/?amp=1
[6] MTA: Mail Transport Agent (SMTP server) https://gitlab.com/muttmua/mutt/-/wikis/MailConcept