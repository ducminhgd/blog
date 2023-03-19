# Don't get hyped - Part 3: Technical stack

> **Don't get hyped** là loạt bài viết nói về những công cụ, công nghệ đang bị được tôn sùng, được ca ngợi quá đà và được áp dụng ồ ạt theo trend, áp dụng vô tội vạ nhưng chưa đúng dẫn đến việc phức tạp hóa vấn đề, dùng dao mổ trâu để giết gà.

Phần này thì mình tập trung vào các công nghệ trên tầng application mà một bộ phận các công ty công nghệ hoặc các đội nhóm đang "đú" theo, đang *tôn thờ*.

> *Lưu ý: trong bài này mình sẽ sử dụng Anh-Việt lẫn lộn vì có một số từ mình không nghĩ ra được cách dịch hay và đúng, nên mình giữ nguyên tiếng Anh*

## Kafka

Với sự *phát triển về mức độ tôn thờ và áp dụng* của Microservices, Event-Driven Design thì phát sinh ra Event Bus và Kafka lại được gọi tên. Nhiều người chỉ dùng Kafka làm một message broker để "giao tiếp" giữa các service của họ, và họ lại đang dựa vào những thông số như số lượng event được xử lý trong một giây rất cao, vân vân và mây mây.

Thật sự các hệ thống mà mình làm có Event-Driven Design hoặc có message queue thì mình chỉ dùng đến RabbitMQ là đủ xài. Theo mình nghĩ, đã áp dụng Event-Driven Design là đã chấp nhận việc bất đồng bộ rồi thì làm cần nhanh đến cả triệu event/giây? Quan trọng là có nhiều hệ thống được áp dụng Kafka vào thì cũng lèo tèo vài chục ngàn event một ngày. Một số đo khác cũng cần được cân nhắc nếu sử dụng Kafka là Transaction Per Second (TPS).

## Go & gRPC

Đầu tiên thì mình xác định là mình không chê bai và cũng không có phủ nhận những thế mạnh của Go. Cái mình đang muốn nói đến là Go không phải một viên đạn bạc để giải quyết mọi vấn đề, đừng ca ngợi quá nhiều. Ngay tại thời điểm mình viết những dòng này thì ở Việt Nam, thị trường tuyển dụng Go đã khắc khe hơn, nhà tuyển dụng đã không cần những bạn trẻ "chỉ mới biết Go" và tuyển với lương khá nữa, họ chuyển sang tuyển các Senior và mức lương cũng vừa phải, không ngáo giá nữa. Ngay tại khoảng thời gian này thì ngôn ngữ đang lên ngôi lại là Java và .NET.

Mình thấy có nhiều bạn ca ngợi là Go nhanh, có goroutine này nọ, xong mình hỏi về goroutine là gì, hoạt động ra sao; process và thread khác nhau sao, hoạt động ra sao; `GOMAXPROCS` là gì thì cũng không biết, vậy mà cứ gọi là biết Go.

Cùng với Go thì gRPC cũng được tôn thờ và xài cùng, có những dự án mới dụng, công ty startup cũng dùng gRPC với lý do là nó nhanh, còn về các điểm trừ của nó thì không ai biết hoặc không đề cập đến. Với lại, mấy cái hệ thống mà vài ngàn user, một ngày có vài chục ngàn request là nhiều mà cứ đòi nhanh làm gì trong khi không tập trung thời gian ra sản phẩm trước, không tập trung giải quyết vấn đề của người dùng, vấn để của doanh nghiệp trước? Đối với mình, gRPC thì chỉ nên áp dụng như những API đã được định nghĩa rõ ràng, chắc chắn và ít thay đổi, chứ nếu đang trong giai đoạn startup thì mình vẫn khuyên nên dùng RESTFul hoặc HTTP JSON API là đủ rồi.

## Database

Mình thấy có nhiều người thích dùng MongoDB hoặc PostgreSQL, khi mình hỏi thì họ lại nói là nó nhanh. Lại là *nhanh*! Có điều nhìn vào thực tế của họ, thì mình thấy họ dùng vì được "tiếp thị" như vậy, ví dụ vì họ xài Odoo, Django thì các bộ open-source đó lại có demo/hướng dẫn hoặc mặc định dùng PostgreSQL, một số open-source khác lại dùng MongoDB. Trong khi đó giờ mình chỉ cần dùng MySQL là có thể xử lý nhanh, dữ liệu nhiều được. Còn nếu cần nhanh, không cần scale, đôi khi mình dùng SQLite thôi là dư sức.

Ngoài ra, mình cần phân biệt là ứng dụng của mình nó đọc nhiều hay ghi nhiều, đọc nhiều thì gắn cache vào, với mình Redis là được, không cần Redis Cluster. Còn nếu ghi nhiều và nếu là bất đồng bộ thì ứng dụng event-sourcing vào và ghi database MySQL là đủ rồi. Mình từng làm qua nhiều hệ thống, có cái CCU lên đến 800k, có cái dữ liệu cả trăm triệu dòng, có cái vài trăm ngàn transaction một ngày thì cũng MySQL và Redis là đủ.

## Kết phần 3

Không có công nghệ nào được gọi là tốt nhất cho mọi trường hợp cả, không có cái gọi là "silver bullet". Một giải pháp, hay một công nghệ nó chỉ có thể đúng hoặc có thể tốt nhất[^1] trong một trường hợp nhất định nào đó, trong một khoảng thời gian nào đó mà thôi.

Gần đây, mình có gia nhập một số hội nhóm (và đã rời bỏ), cứ mỗi lần có người hỏi là giảm latency từ người dùng gọi lên server là hàng trăm câu trả lời *để-tỏ-vẻ-cho-người-khác-biết-là-mình-biết-nhiều* là kêu dùng gRPC, dùng HTTP3/QUIC, tối ưu code, đổi từ PHP hoặc NodeJS hoặc Python sang dùng Go, dùng Rust. Chỉ có duy nhất một trường hợp mà mình thấy trả lời là tối ưu phần network. Cái việc mà các bạn độ xe mạnh cỡ nào mà gặp phải đường xấu thì nó cũng chạy chậm thôi. Đừng có chỉ tin vào những thứ người khác quảng bá mà không có một trải nghiệm thực tế nào cả, hay chí ít cũng nên có một thực nghiệm.

[^1]: tốt nhất mang ý nghĩa là không có gì đúng hay sai, mà là một giải pháp được quyết định bởi sự đánh đổi cái được và cái mất.
