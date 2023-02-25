# Don't get hyped - Part 2: Architecture

Ở phần này mình sẽ nói về Microservices, một thiết kế mà nhiều người đang ca tụng và làm theo những cách sai lầm khác nhau.

> *Lưu ý: trong bài này mình sẽ sử dụng Anh-Việt lẫn lộn vì có một số từ mình không nghĩ ra được cách dịch hay và đúng, nên mình giữ nguyên tiếng Anh*

## Không độc lập

Đa số lý do mà đa số người hay nói về *Tại sao nên dùng Microservices* đó là *nếu một service bị chết thì các service khác vẫn chạy bình thường*. Và OK! Nó đúng! Rất nhiều service mà mình tham gia điều tra hậu sự cố thì đúng là *các service khác vẫn chạy bình thường* **nhưng không hoạt động bình thường**. Nghĩa là instance/pod/VM của các service liên quan vẫn chạy, healthcheck vẫn tốt và không có request gọi vào, nếu có là phát sinh lỗi. Và đâu đó, dấu hiệu này thường đi kèm là khi deploy services lúc ban đầu, tất cả phải deploy cùng lúc, không có cái lên trước lên sau.

## Boundary & scope

Việc phân tách thành Microservices phải dựa trên những tiêu chí về nguồn lực hoặc domain của các services. Phân chia không rõ ràng, dẫn đến việc service này phải quản lý dữ liệu, hoặc chứa dữ liệu trùng, hoặc có một phần logic nghiệm vụ của service khác hoặc phải nhân bản service này ra thành service khác. Ví dụ: hệ thống có một User Service chuyên dùng để quản lý người dùng, phân quyền; một ngày nào đó, do nhu cầu phát triển mà có quản lý người dùng hệ thống, người dùng cuối và kể cả đối tác thì vô tình bị clone ra thành Admin User Service, User Service và Partner Service; trong khi cả 3 service đều làm chung nhiệm vụ nhưng lại do cho các nhóm đối tượng khác nhau, dẫn đến việc các service khác sử dụng dịch vụ với cùng một logic nhưng phải phân biệt khi nào nên gọi service nào.

## Services quá nhỏ

Do một số người có suy nghĩ là "Microservice phải nhỏ" nhưng nhỏ bao nhiêu là đủ thì lại không xác định được, do việc phân tách làm không đúng, không chia phạm vi rõ ràng và kèm theo tư tưởng là Microservice thì dễ dàng tích hợp, dẫn đến việc đẻ ra cho một đống nanoservice chỉ là những việc lặt vặt mà xem nó hẳn là microservice, tốn thêm nguồn lực để phát triển và duy trì. Viễc này góm phần tăng độ phức tạp lên.

## Design Patterns

Một vấn đề khác nữa là về thiết kế. Thường thấy có hai hướng sai là: sử dụng không đúng thiết kế hoặc là chế cháo cái thiết kế trời ơi đất hỡi. Ví dụ, nghe một cái thuộc ngữ là `Event-Driven Design` là cái gì cũng tạo thành event, rồi đẻ ra cái gọi là event bus, sau đó là theo kiểu đồng bộ (synchronous) thông qua event bus đó. Thật sự, muốn làm gì thì mình nghĩ là lên đọc hết các pattern trên https://microservices.io là đủ để làm rồi.

## Logging và database độc lập

Việc tách ra làm Microservices không có nghĩa là bắt buộc phải có database độc lập, trong các design patterns của Microservices thì vẫn có trường hợp chia sẻ database với nhau. Việc có database độc lập thường phát sinh ba vấn đề cần giải quyết là trung lặp dữ liệu, nguồn dữ liệu đúng (single source of truth) và tương tác giữa các service.

Ngoài ra, việc ghi log cũng vậy, nên ghi tập trung hơn là ghi lẻ tẻ từng service. Mình từng thấy có một số Microservice được dựng lên và ghi log vào database, mà database thì lại tách rời nhau, dẫn đến việc log nhiều, chết database và log không tập trung, mỗi lần truy vết phải đi lang thang mà lượm lặt.

Đề xuất của mình thì chẳng có gì lạ, cứ như nhiều người làm thôi: xây dựng nơi bắn dữ liệu log, dữ liệu cần thu thập về một chỗ. Đừng có tách rải rác rồi rồi có service đi lượm rác lại, không cần thiết lắm.

## Có thật sự cần thiết

Cuối cùng, như chủ đề của loạt bài viết này, có thật sự cần thiết dùng Microservices cho hệ thống của chính bạn không? Ví dụ, một trang web bán hàng của bạn, một ngày khoảng 1000 đơn hàng, không có ý định mở rộng thêm, không có nhu cầu đưa thành một PaaS hay SaaS thì liệu có cần áp dụng Microservices?

Microservices được áp dụng vào không mang đến giá trị đơn giản hóa mà là phức tạp hóa, nó bắt buộc phải mở rộng về mặt công việc và nhân sự. Do đó, áp dụng không đúng lúc, không đúng chỗ thì chỉ mang tính chất hao phí đầy rủi ro.
