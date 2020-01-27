---
title: 使用 gomail 发送邮件
date: 2020-01-27 21:11:21
tags:
- Go
categories:
- Go
---
# 使用 gomail 发送邮件

使用 gomail 的一些记录笔记。

## 准备工作
* 准备 SMTP/IMAP 服务用于代发邮件（这部分内容不赘述，网上可以搜索关键字：QQ/163/gamil SMTP即可）
* 写邮件需要填写哪些信息
  * From/发件人（必须）
  * To/收件人（必须）
  * Cc/抄送（可选）
  * Subject/标题（必须）
  * Body/邮件正文内容（必须）
  * Attach/附件（可选）

## 使用 gomail
* 下载 gomail 包

```go
go get gopkg.in/gomail.v2
```

* 创建一个要发送邮件的对象

```
m ：= gomail.NewMessage()
```

gomail 中使用 NewMessage 方法创建一个新的消息对象，默认是使用 UTF-8 字符集的，在这个创建的消息对象上设置发件人、收件人等邮件的信息，所以下面填写邮件信息的操作都是基于 NewMessage 方法创建的对象基础上的。

* 填写 From/发件人

```go
m.SetHeader("From","from@xx.com")
```

gomail 中通过 `func (m *Message) SetHeader(field string, value ...string)` 方法设置收件人、发件人、标题等信息，SetHeader 方法第一个参数表示设定要设置的内容，From 表示发件人，To 表示收件人，Subject 表示标题，后面可以跟多个 String 类型的参数，可以表示多个收件人


* 填写 To/收件人

```go
m.SetHeader("To","to@xx.com","to_1@xx.com")
```

* 填写 Cc/抄送人

```go
m.SetAddressHeader("Cc","cc@xx.com","cc")
```

gomail 中通过 `func (m *Message) SetAddressHeader(field, address, name string)` 方法设置抄送人，SetAddressHeader 方法第一个参数表示设定要设置的内容，第二个参数表示抄送人的邮箱地址，第三个参数表示抄送人名称


* 填写 Subject/标题

```go
m.SetHeader("Subject","This is mail title")
```

* 填写 Body/邮件正文内容

```go
m.SetBody("text/html","Mr.Li<br>Thanks For your help.")
```

gomail 中通过 `func (m *Message) SetBody(contentType, body string, settings ...PartSetting)` 方法设置邮件正文内容， SetBody 方法第一个参数表示设置文本内容类型，可以是 text/plain 也可以是 text/html，第二个参数表示正文内容字符串，第三个参数表示一些额外的设置（大部分情况可以不设置），如果有字符 encode 的需要可以设置

* 添加附件 

```go
m.Attach("../../go.mod")
```

gomail 中通过 `func (m *Message) Attach(filename string, settings ...FileSetting)` 方法添加邮件附件，Attach 方法第一个参数表示附件文件路径，第二个参数表示表示一些额外的设置，比如附件文件名称修改等（可以不设置）

* 设置 SMTP/IMAP 服务器

```go
d := gomail.NewDialer("smtp.qq.com",465, "1234567890@qq.com", "xxxxxxxx")
```

gomail 中通过 `func NewDialer(host string, port int, username, password string) *Dialer` 方法创建一个 SMTP Dialer 用于发送邮件

* 发送邮件

```go
err := d.DialAndSend(m)
if err != nil{
    panic(err)
}
```

gomail 中通过 `func (d *Dialer) DialAndSend(m ...*Message) error` 方法连接到 SMTP 服务器并发送邮件，最终关闭连接。

* 使用 SetHeaders 方法一起设置邮件内容

前面介绍设置邮件发件人、收件人、标题等信息都是通过 SetHeader 方法，调用 SetHeader 方法只能设置一项内容，如果想要同时设置多项，那可以考虑使用 SetHeaders 方法，传入一个 `map[string][]string` 类型的参数即可


```go
mailHeader := map[string][]string{
		"From":    {"from@qq.com"},
		"To":      {"to_user1@qq.com", "to_user2@gmail.com"},
		"Subject": {"标题"},
	}
m.SetHeaders(mailHeader)
```

## 代码示例 

``` go
import (
	"gopkg.in/gomail.v2"
)

func BaseSend() {
	mailHeader := map[string][]string{
		"From":    {"from@qq.com"},
		"To":      {"to_user1@qq.com", "to_user2@gmail.com"},
		"Subject": {"标题"},
	}

	m := gomail.NewMessage()
	m.SetHeaders(mailHeader)
	m.SetAddressHeader("Cc", "shengang@pingcap.com", "shengang")
	m.SetBody("text/html", "尊敬的客户")
	m.Attach("../../go.mod")

	d := gomail.NewDialer("smtp.qq.com", 465, "1234567890@qq.com", "xxxxxxxx")

	if err := d.DialAndSend(m); err != nil {
		panic(err)
	}
}
```


## 参考链接
* gomail Github:https://github.com/go-gomail/gomail
* gomail example:https://godoc.org/gopkg.in/gomail.v2

