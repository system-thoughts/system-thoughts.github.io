给社区发送邮件讨论补丁时，选择一个趁手的客户端会极大提高沟通效率。曾经我通过"git send-email + thunderbird"的工具组合进行邮件的发送和读取，[thunderbird如同outlook的clone版本](https://www.kernel.org/doc/html/v4.10/process/email-clients.html)，对于使用GUI客户端的用户极为友好。然而大多数内核开发者还是会选用mutt邮件客户端，其可配置性极佳，完全命令行的交互方式对于kernel hacker来说是极为友好的。

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

mutt中的变量通过`set var=value`方式配置。存在仅需设置yes/no的`toggle var`（布尔变量），你可以为`toggle var`设置"ask-yes"或者"ask-no"以指定每次使用时给定默认答案进行提示。也可以通过 "unset"、"set var"、"set novar"简化布尔变量的设置。需要注意的是，你配置的mutt变量对应的功能在编译mutt时已启用才会成功设置，否则你会得到“unknown variable”的告警。下面给出最基本的mutt配置：

`ssl_force_tls`表示mutt与所有的远端服务器的连接需要通过TLS协议加密，如果不能成功建立连接，中断通信。需要通过`mutt -v | grep tls`能够打印`--with-gnutls`表示mutt支持TLS。

`abort_nosubject`的默认值是"ask-yes"，表示我们发送邮件还未写邮件主题时，该配置将会提示，默认值为"yes"。将该项设置为"no"，便无需确认。

mutt每次接收到键盘输入时便会更新所有目录的状态。我们也希望即使在空闲时也能收到新邮件的通知，而不需要按下按键。控制这种行为的变量是`timeout`。它表示等待用户输入的最大时间，以秒为单位。如果在指定的时间内没有接收到用户输入，便执行更新操作。变量的默认值是600秒，表示在没有输入的情况下，每10分钟接收一次更新。默认值太高，我们设置为10。

如前所述，每次收到用户输入时，mutt都会查找更新。键盘活动太频繁会导致过多的访问操作，为了限制这个频率，使用`mail check`变量。该变量表示两次扫描之间的最小时间(以秒为单位)。该变量的默认值是5，即使经常按下键，mutt也将每5秒搜索一次新邮件。该值还是太小，尤其是在多邮箱的场景下，可能因为频繁访问降低速度。

默认情况下，索引菜单(显示消息列表)中的电子邮件按日期升序排序，因此更新的电子邮件将显示在底部。要更改电子邮件的排序方式，我们可以使用和设置`sort`变量的值。在本例中，我们设置`reverse-data-received`让更新的电子邮件出现在列表的顶部。其他参数也可以用作排序因子，例如subject和size。


## Retrieve email
MUA使用POP3/IMAP协议从邮件服务器下载邮件。两者的区别如下[3]：
* POP3(Post Office Protocol 3)：仅仅下载邮件服务器的inbox目录中的邮件到本地，不会下载sent、draft、deleted目录下的邮件。并且POP3不会同步，并且当email被下载到一台设备上时，email会从邮件服务器中删除。
* IMAP(Internet Message Access Protocol)：允许在多个客户端上查看邮件（同步），IMAP会将邮件缓存到本地。IMAP还会同步各个设备的目录结构

{% asset_img POP3.png %}
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

mutt已经支持IMAP，验证当前版本mutt是否支持IMAP：
```bash
[root@localhost ~]# mutt -v | grep IMAP
+USE_POP  +USE_IMAP  +USE_SMTP  
```
显然，当前mutt版本已经支持IMAP。那么，我们直接使用mutt自带的IMAP功能下载邮件。



## Send email
MTA进行邮件转发使用SMTP协议(Simple Mail Transfer Protocol)。如下这段话，我认为是对SMTP协议的功能的一个良好概括[4]。
> SMTP is basically a set of commands that authenticates and directs the transfer of email

更简单方式的记住S(ending) M(ail) T(o) P(eople)。下图展示了通过SMTP协议将邮件从SMTP client途径SMTP server传递到收件人的SMTP server，最终收件用户登录信箱通过POP3/IMAP下载邮件。
{% asset_img smtp.png %}

MTA可以分为两类：仅转发(relay-only)、全功能(full-fledged)[5]。
* Relay-only MTAs: 或者称为Send-only MTAs、small MTAs(SMTP clients)[6]，此类MTA仅将你的email转发到另一个服务器，如果你像我一样仅仅想发送自己的Gmail邮件，此类MTA是最好的选择。SMTP client仅执行某些特定的功能，而不像Full-fledged MTA一样运行完成的SMTP服务器，占用大量的资源开销。这样的MTA不会监听传入的消息，尽在需要发送邮件的时候运行。small MTA通常作为MSA即邮件发送的第一站。
* Full-fledged MTAs：也称为mail hub，能够处理Internet邮件传送的所有细节，通常这类MTA从Relay-Only MTA接收邮件再进行转发。邮件在Internet的中间MTA转发漫游时，这些MTA使用full-fledged MTA比较合适。

本文仅考虑MSA即Relay-only MTA，选取软件作为MSA应该考虑如下因素：
* MTA是否可以对邮件进行排队，以便在出现故障时稍后发送
* MTA可以取代MDA?如果可以，它将处理来自系统的所有邮件。
* MTA是否支持连接到ISP SMTP服务器的要求？这些要求可能包括特定的身份验证或TLS。

mutt早已支持ESMTP/SMTP，验证当前您使用的mutt是否支持SMTP:
```bash
[root@localhost ~]# mutt -v | grep SMTP
+USE_POP  +USE_IMAP  +USE_SMTP
```
显然，当前mutt版本已经支持SMTP。那么，我们直接使用mutt自带的SMTP功能进行邮件发送，此时，mutt自己可以作为MSA。如果您需要选择功能更为复杂的MSA，请参考[SendmailAgents](https://gitlab.com/muttmua/mutt/-/wikis/SendmailAgents)。



## Reference
[1] Email agent (infrastructure) https://en.wikipedia.org/wiki/Email_agent_(infrastructure)#cite_note-modularmonolithic-schroder-1
[2] SendmailAgents https://gitlab.com/muttmua/mutt/-/wikis/SendmailAgents
[3] POP3 vs IMAP - What's the difference? https://www.youtube.com/watch?v=SBaARws0hy4&t=337s
[4] What is SMTP - Simple Mail Transfer Protocol https://www.youtube.com/watch?v=PJo5yOtu7o8
[5] Postfix vs. Sendmail vs. Exim https://blog.mailtrap.io/postfix-sendmail-exim/?amp=1
[6] MTA: Mail Transport Agent (SMTP server) https://gitlab.com/muttmua/mutt/-/wikis/MailConcept