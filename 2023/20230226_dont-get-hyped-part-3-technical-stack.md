# Don't get hyped - Part 3: Technical stack

Phần này thì mình tập trung vào các công nghệ trên tầng application mà một bộ phận các công ty công nghệ hoặc các đội nhóm đang "đú" theo, đang *tôn thờ*.

## Kafka

Với sự *phát triển về mức độ tôn thờ và áp dụng* của Microservices, Event-Driven Design thì phát sinh ra Event Bus và Kafka lại được gọi tên. Nhiều người chỉ dùng Kafka làm một message broker để "giao tiếp" giữa các service của họ, và họ lại đang dựa vào những thông số như số lượng event được xử lý trong một giây rất cao, vân vân và mây mây.

Thật sự các hệ thống mà mình làm có Event-Driven Design hoặc có message queue thì mình chỉ dùng đến RabbitMQ là đủ xài. Theo mình nghĩ, đã áp dụng Event-Driven Design là đã chấp nhận việc bất đồng bộ rồi thì làm cần nhanh đến cả triệu event/giây? Quan trọng là có nhiều hệ thống được áp dụng Kafka vào thì cũng lèo tèo vài chục ngàn event một ngày.

## Go & gRPC

Đầu tiên thì mình xác định là mình không chê bai và cũng không có phủ nhận những thế mạnh của Go. Cái mình đang muốn nói đến là Go không phải một viên đạn bạc để giải quyết mọi vấn đề, đừng ca ngợi quá nhiều. Ngay tại thời điểm mình viết những dòng này thì ở Việt Nam, thị trường tuyển dụng Go đã khắc khe hơn, nhà tuyển dụng đã không cần những bạn trẻ "chỉ mới biết Go" và tuyển với lương khá nữa, họ chuyển sang tuyển các Senior và mức lương cũng vừa phải, không ngáo giá nữa. Ngay tại khoảng thời gian này thì ngôn ngữ đang lên ngôi lại là Java và .NET.

Mình thấy có nhiều bạn ca ngợi là Go nhanh, có goroutine này nọ, xong mình hỏi về goroutine là gì, hoạt động ra sao; process và thread khác nhau sao, hoạt động ra sao; `GOMAXPROCS` là gì thì cũng không biết, vậy mà cứ gọi là biết Go.

Cùng với Go thì gRPC cũng được tôn thờ và xài cùng, có những dự án mới dụng, công ty startup cũng dùng gRPC với lý do là nó nhanh, còn về các điểm trừ của nó thì không ai biết hoặc không đề cập đến. Với lại, mấy cái hệ thống mà vài ngàn user, một ngày có vài chục ngàn request là nhiều mà cứ đòi nhanh làm gì trong khi không tập trung thời gian ra sản phẩm trước, không tập trung giải quyết vấn đề của người dùng, vấn để của doanh nghiệp trước? Đối với mình, gRPC thì chỉ nên áp dụng như những API đã được định nghĩa rõ ràng, chắc chắn và ít thay đổi, chứ nếu đang trong giai đoạn startup thì mình vẫn khuyên nên dùng RESTFul hoặc HTTP JSON API là đủ rồi.

## Database

Mình thấy có nhiều người thích dùng MongoDB hoặc PostgreSQL, khi mình hỏi thì họ lại nói là nó nhanh. Lại là *nhanh*! Có điều nhìn vào thực tế của họ, thì mình thấy họ dùng vì được "tiếp thị" như vậy, ví dụ vì họ xài Odoo, Django thì các bộ open-source đó lại có demo/hướng dẫn hoặc mặc định dùng PostgreSQL, một số open-source khác lại dùng MongoDB. Trong khi đó giờ mình chỉ cần dùng MySQL là có thể xử lý nhanh, dữ liệu nhiều được. Còn nếu cần nhanh, không cần scale, đôi khi mình dùng SQLite thôi là dư sức.

Ngoài ra, mình cần phân biệt là ứng dụng của mình nó đọc nhiều hay ghi nhiều, đọc nhiều thì gắn cache vào, với mình Redis là được, không cần Redis Cluster. Còn nếu ghi nhiều và nếu là bất đồng bộ thì ứng dụng event-sourcing vào và ghi database MySQL là đủ rồi. Mình từng làm qua nhiều hệ thống, có cái CCU lên đến 800k, có cái dữ liệu cả trăm triệu dòng, có cái vài trăm ngàn transaction một ngày thì cũng MySQL và Redis là đủ.
