# Gưi email trong Go

Mặc dù mình có tạm message với các *headers* cho *To*, *CC*, *BCC* nhưng đối với function `smtp.SendMail`, nếu mình sử dụng `to` thay cho `rcpts` nội dung email vẫn có đủ *To*, *CC*, *BCC* và email chỉ được gửi đến những địa chỉ email có trong biến `to` chứ không gửi đến những người có trong biến `cc` và `bcc`

```go
func SendEmailMessage(to, cc, bcc, body, subject string) bool {
	port := Getenv("SMTP_PORT", "587")
	smtpServerHost := Getenv("SMTP_SERVER", "smtp.gmail.com")
	senderEmail := Getenv("SMTP_USERNAME", "")
	senderPassword := Getenv("SMTP_PASSWORD", "")
​
	if smtpServerHost == "" || senderEmail == "" || senderPassword == "" {
		log.Println("Error: sender info is required")
		return false
	}
​
	headers := "From: " + senderEmail
	headers = headers + "\nTo: " + to + ";"
	rcpts := to
	if len(cc) > 0 {
		headers += "\nCc: " + cc + ";"
		rcpts += "," + cc
	}
	if len(bcc) > 0 {
		headers += "\nBcc: " + bcc + ";"
		rcpts += "," + bcc
	}
	headers = strings.ReplaceAll(headers, ",", ";")
​
	smtpServer := smtpServerHost + ":" + port
	mime := "Content-Type: text/html; charset=UTF-8"
	body = string(markdown.ToHTML([]byte(body), nil, nil))
	msg := []byte(headers + "\nSubject: " + subject + "\n" + mime + "\n" + body + "\n")
	auth := smtp.PlainAuth("", senderEmail, senderPassword, smtpServerHost)
	err := smtp.SendMail(smtpServer, auth, senderEmail, strings.Split(rcpts, ","), msg)
​
	if err != nil {
		log.Printf("Error:  %v", err)
		return false
	}
	return true
}
```