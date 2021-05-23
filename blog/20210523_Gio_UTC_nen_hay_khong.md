# Giờ UTC có nên dùng hay không?

Bài viết này xuất phát từ việc mình phải thuyết phục các anh em trong công ty hiện tại chuyển giờ của Server và Database về UTC, do đó mình phải tìm hiểu lại để có thể thuyết phục họ hay chính xác là thuyết phục được mình là _mình vẫn đang đi hướng đi phù hợp_.

Sau khi đọc một mớ bài viết thì mình nhận ra rằng, UTC có thể xem là một _định dạng thời gian_ chứ không phải là múi giờ +00:00, việc viết UTC mà không ghi rõ múi giờ thì _mặc định là +00:00_, đây là điề mình luôn lầm tưởng đó giờ. Lợi ích của dùng UTC là nó không quan tâm múi giờ, nó chỉ biết so sánh thời gian của múi giờ +00:00 đến vị trí (offset) của múi giờ cần so sánh. Do không quan tâm timezone nên cũng bỏ qua luôn Daylight Saving Time (DST - việc tăng giảm 1 giờ vào một số ngày nhất định trong năm ở một số khu vực trên thế giới)

Do đó:
1. Database nên lưu trữ giờ UTC+00:00 để dễ convert ra UTC hoặc ISO8601. Và dễ dàng migrate database sang các databas server khác, đồng nhất giữa các service cho việc chuyển đổi thời gian.
1. Dockerfile không nên có những đoạn script đổi múi giờ trong đó.
1. Service ở mỗi vùng có thể có múi giờ khác nhau.
1. APIs, Web services và các hệ thống nên dùng format input và output là ISO8601 để tiện cho việc chuyển đổi và hiển thị thời gian.

## Link tham khảo

1. https://zachholman.com/talk/utc-is-enough-for-everyone-right
    Ở bài viết này mình học được một số điều sau:
    - Lịch sử phát triển _thời gian_ hay là lịch sử mà con người quy định thời gian.
    - [Swatch Internet Time](https://en.wikipedia.org/wiki/Swatch_Internet_Time).
    - [tz database](https://en.wikipedia.org/wiki/Tz_database) còn được gọi là _Olson database_.
    - Trích nguyên văn _Timezones are weird. Time is weird._
    - [Ngày 30 tháng 2 có tồn tại](https://www.timeanddate.com/date/february-30.html)
    - Đồng hồ không có số 0 vì trong số La Mã không có ký tự nào đại diện cho số 0
    - Download được [Whitepaper về Recurring của Martin Fowler](https://martinfowler.com/apsupp/recurring.pdf)
1. https://kylekatarnls.medium.com/always-use-utc-dates-and-times-8a8200ca3164