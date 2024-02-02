---
title: "Send E-Mails with Go using net/smtp package"
date: 2024-02-02T13:29:25+01:00
draft: false
cover: ""
tags: [
    "go",
    "golang",
    "smtp",
]
categories: [
    "Software",
]
description: "Embed a single page application in Golang using Gin framework (using a single http server)."
---

Working with e-mails with Go is extremely easy, even with a topic so daunting as it.  
We'll leaverage the `net/http` and `bytes` packages provided by the Go standard library.

## Configurations

First of all we need to have the credentials to our smtp server.  
Beware that sending emails like this can trigger the spam detection on some providers.
After that some users report your emails as non-spam they'll not be marked as such.

```go
package mailer

import (
	"bytes"
	"fmt"
	"html/template"
	"mime/multipart"
	"net/smtp"
	"os"
)

var (
	from     = os.Getenv("SMTP_USER") 
	pass     = os.Getenv("SMTP_PASS")
	smtpHost = os.Getenv("SMTP_HOST")
	smtpPort = os.Getenv("SMTP_PORT")
	alias    = os.Getenv("SMTP_ALIAS")
	smtpAddr = smtpHost + ":" + smtpPort
	auth     = smtp.PlainAuth(alias, from, pass, smtpHost)
)
```

## Sending emails with html template as body

Using the `html/template` package the email body can be set as a html document 
leaveraging the whole power of go templating. Also, the emails will be as pretty as you want 🤗

```go
...

func SendMail(data *internal.Registration, tmpl *html.Template) error {
  // initialize the shared buffer
  var buf bytes.Buffer
	
  // set the email subject
	buf.WriteString("Subject: Confirm your registration\r\n")
  // optionally, set the carbon copy
	buf.WriteString("Cc: other@example.org\r\n")
  // set the mime type as defined in RFC2045
  // https://datatracker.ietf.org/doc/html/rfc2045.html
	buf.WriteString("MIME-Version: 1.0\n")
	buf.WriteString("Content-Type: text/html; charset=utf-8;\n")
	
  // copy the template html output to the shared buffer
  tmpl.Execute(&buf, data)

  // terminate the message
	buf.WriteString("\n\n")

  to := []string{data.Email}
  // send the resulting buffer with the `net/smtp` package
	return smtp.SendMail(smtpAddr, auth, from, to, buf.Bytes())
}
```
Done! Simply as that. I want to reiterate on what said before. This does 
not handle complex auth systems and message validation strageties so it can
trigger spam protection.

## Sending emails with html template as body and attachment

Hold on! How can I handle attachments?  
We need to handle the message as a multipart message 
(as defined in [RFC1314](https://www.w3.org/Protocols/rfc1341/7_2_Multipart.html))

As an attachment i will provide a real world example, an ICS calendar.  

If you're interested I'll covering my ICS library in this [article](/posts/golang/golang-ics/).
```go
...

func SendMail(data *internal.Booking, tmpl *html.Template) error {
	var (
		buf      bytes.Buffer                 // shared buffer
		writer   = multipart.NewWriter(&buf)  // handles multipart boundaries
		boundary = writer.Boundary()          // boundary for-each message part
	)

	// set the email subject
	buf.WriteString("Subject: Confirm your registration\r\n")
  // optionally, set the carbon copy
	buf.WriteString("Cc: other@example.org\r\n")
  // set the mime type as defined in RFC2045
  // https://datatracker.ietf.org/doc/html/rfc2045.html
	buf.WriteString("MIME-Version: 1.0\n")
  // set the boundary ref
	buf.WriteString(fmt.Sprintf("Content-Type: multipart/mixed; boundary=%s\n", boundary))
  // set the first boundary
	buf.WriteString(fmt.Sprintf("--%s\n", boundary))

  // set the main part mime type
	buf.WriteString("Content-Type: text/html; charset=utf-8;\n")
  // execute the template and write the output to the shared buffer
	tmpl.Execute(&buf, data)

  // end the first part with the boundary
	buf.WriteString(fmt.Sprintf("\n\n--%s\n", boundary))

  // generate an ics calendar and write the output to the shared buffer
	ics.AsAttachment(&buf, &ics.ICSData{
		Id:               "PRODID:-//Hello//EN",
		Uid:              data.Id,
		ISODateCreatedAt: ics.FormatDate(time.Now().Format(time.RFC3339)),
		ISODateStart:     ics.FormatDate(data.Date.Format(time.RFC3339)),
		ISODateEnd:       ics.FormatDate(data.Date.Add(time.Minute * 30).Format(time.RFC3339)),
		Organizer:        "Marco",
		Summary:          "Meeting",
		Address:          "Elune",
		Sender:           "mailto:hello@example.org",
		Email:            data.Email,
	})

  // terminate the second part
	buf.WriteString(fmt.Sprintf("\n--%s", boundary))
  // terminate the message
	buf.WriteString("--")

  // send
	return smtp.SendMail(smtpAddr, auth, from, []string{data.Email}, buf.Bytes())
}
```

`AsAttachment` essentially does...

```go
...
buf.WriteString("Content-Type: text/calendar; charset=utf-8; method=REQUEST\n")
buf.WriteString("Content-Transfer-Encoding: 7bit\n")
// execute
buf.WriteString("\r\n")
buf.WriteString(data)
buf.WriteString("\r\n")  
...
```

You can add as many attachments by repeating the pattern: 
**boundary, data, newline, --boundary--, newline.**

## Conclusions

As inherent to the Golang philosophy, even sending emails is super easy and 
pretty bulletproof.