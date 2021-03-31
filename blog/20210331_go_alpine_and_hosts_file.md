# Go, Alpine & hosts file

## Vấn đề 

Team mình đang xây dựng một tính năng gửi email bằng Go với đoạn code như sau:

```go
package tools

import (
	"log"
	"net/smtp"
	"os"
	"strings"

	"github.com/gomarkdown/markdown"
	"github.com/subosito/gotenv"
)

func init() {
	gotenv.Load()
}

func Getenv(key, fallBack string) string {

	value := os.Getenv(key)
	if len(value) == 0 {
		return fallBack
	}
	return value
}

func SendEmail(to, cc, bcc, body, subject string) bool {
	port := Getenv("SMTP_PORT", "587")
	smtpServerHost := Getenv("SMTP_SERVER", "smtp.gmail.com")
	senderEmail := Getenv("SMTP_USERNAME", "")
	senderPassword := Getenv("SMTP_PASSWORD", "")

	if smtpServerHost == "" || senderEmail == "" || senderPassword == "" {
		return false
	}

	headers := "From: " + senderEmail
	headers = headers + "\nTo: " + to
	if len(cc) > 0 {
		headers += "\nCc: " + cc
	}
	if len(bcc) > 0 {
		headers += "\nBcc: " + bcc
	}
	headers = strings.ReplaceAll(headers, ",", ";")

	smtpServer := smtpServerHost + ":" + port
	mime := "Content-Type: text/html; charset=UTF-8"
	body = string(markdown.ToHTML([]byte(body), nil, nil))
	msg := []byte(headers + "\nSubject: " + subject + "\n" + mime + "\n" + body + "\n")
	auth := smtp.PlainAuth("", senderEmail, senderPassword, smtpServerHost)
	err := smtp.SendMail(smtpServer, auth, senderEmail, strings.Split(to, ","), msg)

	if err != nil {
		log.Println(err)
		return false
	}
	return true
}
```

`Dockerfile` để build Docker image:

```Dockerfile
FROM golang:1.15-alpine as builder

ENV GOARCH="amd64" \
    GOOS=linux

WORKDIR /app
COPY . .
RUN go build -tags musl --ldflags "-extldflags -static"  -o gitlab .
RUN apk add --update curl && curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl \
    && chmod +x ./kubectl \
    && mv ./kubectl /usr/local/bin

FROM alpine:3.13.2

WORKDIR /
COPY --from=builder /app/gitlab /usr/local/bin/
COPY --from=builder /usr/local/bin/kubectl /usr/local/bin/kubectl
```

Do dùng mail server công ty không mở public nên phải dùng thông qua proxy, mình chạy câu lệnh `docker run --rm --add-host=stmp.company.example:smtp.my-proxy.example image-name:1.0.0 send-email` thì container vẫn gọi trực tiếp vào `smtp.company.example` chứ không gọi thông qua proxy.

_Chú thích:_ 
- `image-name` là tên image mình build ra.
- `send-email` là command gọi function `SendEmail` đã được xử lý và viết trong file `main.go` của dự án.
- Một số tham số liên quan nhưng không quan trọng đã được bỏ ra khỏi command ví dụ.

## Tìm hiểu và giải quyết

Sau khi đọc nhiều link trên mạng thì nguyên nhân có thể được mô tả vắn tắt như sau:

The Name Service Switch (NSS) configuration file, `/etc/nsswitch.conf`, được sử dụng bởi GNU C Library và một vài ứng dụng khác để xác định nguồn của việc phân giải tên và thứ tự sử dụng các nguồn đó. `musl` và `glibc` là hai thư viện C, các ứng dụng Go thường được build bằng hai thư viện này. `glibc` có tính tolerance cao, nên sẽ có trường hợp failed-over cho việc không tìm thấy file `/etc/nsswitch.conf`, còn `musl` thì không.

Alpine là `musl`-based linux và trong image Alpine đã không tồn tại file `/etc/nsswitch.conf`, cộng với việc build Go bằng thư viện `musl` (`go build -tags musl --ldflags "-extldflags -static"  -o gitlab .`) nên ứng dụng Go được build ra thì tìm kiếm file `/etc/nsswitch.conf` không thấy và dừng lại, dẫn đến việc ứng dụng không đọc file `/etc/hosts`. Với Python thì sẽ đọc file `/etc/hosts`.

Giải pháp là tạo ra file `/etc/nsswitch.conf` và cấu hình cho đọc Name-service từ file `/etc/hosts`. Thêm dòng `RUN echo 'hosts: files dns' > /etc/nsswitch.conf` vào cuối Dockerfile

```Dockerfile
FROM golang:1.15-alpine as builder

ENV GOARCH="amd64" \
    GOOS=linux

WORKDIR /app
COPY . .
RUN go build -tags musl --ldflags "-extldflags -static"  -o gitlab .
RUN apk add --update curl && curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl \
    && chmod +x ./kubectl \
    && mv ./kubectl /usr/local/bin

FROM alpine:3.13.2

WORKDIR /
COPY --from=builder /app/gitlab /usr/local/bin/
COPY --from=builder /usr/local/bin/kubectl /usr/local/bin/kubectl

RUN echo 'hosts: files dns' > /etc/nsswitch.conf
```

## Các links đọc Thêm

1. https://github.com/golang/go/issues/35305
1. https://github.com/gliderlabs/docker-alpine/issues/367
1. https://github.com/docker-library/docker/issues/82#issuecomment-334627834
1. https://ops.tips/notes/glibc-golang-learnings/
1. https://github.com/golang/go/blob/4bd95702dd1e81f383ee67c14945620d30247908/src/net/conf.go#L219-L253