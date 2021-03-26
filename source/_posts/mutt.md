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


## Send email
MTA可以分为两类：仅转发(relay-only)、全功能(full-fledged)[2]。
* Relay-only MTAs: 或者称为Send-only MTAs、small MTAs(SMTP clients)[3]，此类MTA仅将你的email转发到另一个服务器，如果你像我一样仅仅想发送自己的Gmail邮件，此类MTA是最好的选择。SMTP client仅执行某些特定的功能，而不像Full-fledged MTA一样运行完成的SMTP服务器，占用大量的资源开销。这样的MTA不会监听传入的消息，尽在需要发送邮件的时候运行。small MTA通常作为MSA即邮件发送的第一站。
* Full-fledged MTAs：也称为mail hub，能够处理Internet邮件传送的所有细节，通常这类MTA从Relay-Only MTA接收邮件再进行转发。邮件在Internet的中间MTA转发漫游时，这些MTA使用full-fledged MTA比较合适。

## Reference
[1] Email agent (infrastructure) https://en.wikipedia.org/wiki/Email_agent_(infrastructure)#cite_note-modularmonolithic-schroder-1
[2] Postfix vs. Sendmail vs. Exim https://blog.mailtrap.io/postfix-sendmail-exim/?amp=1
[3] MTA: Mail Transport Agent (SMTP server) https://gitlab.com/muttmua/mutt/-/wikis/MailConcept




* MSA(Message Submission Agent)：接收来自MUA发送的邮件，同MTA合作进行邮件转发




最细粒度的划分：
* MUA(Message User Agent)


* 
* MRA(Mail Retrieval Agent)(与上述术语不同，)

MSA使用SMTP协议的变种ESMTP协议，MTAs基本上都具备MSA的功能，很少有程序是专门为MSA设计而不具备完整的MTA功能的。MTA和MSA功能都使用端口号25，但是MSA的官方端口是587。MTA接收用户的外来邮件(incoming email)，而MSA接收用户的外发邮件(outgoing email)。从这个细微的差异化描述来看，MSA仅能作为MUA外发邮件的第一站，而MTA可以作为邮件传送过程的任一中转站。
MSA和MTA的功能分离带来了以下的好处：
* MSA可在邮件发送之前向作者报告邮件存在的错误，因为MSA直接和MUA交互，因此它可以纠正并报告消息格式中的小错误（例如缺少日期、消息ID、收件人字段或缺少域名的地址），以便作者可以在邮件发送出去之前进行更正。而MTA仅能在邮件送达之后才能报告错误。
* MSA有专用端口号587，用户总是可以连接到他们的域(domain)来提交新邮件。为了反制垃圾邮件，许多ISP和机构网络限制了通过25端口连接远程MTA的能力。MSA在587端口上的可访问性是的笔记本用户即使在其他网络中也可以通过首选的提交服务器发送邮件。
