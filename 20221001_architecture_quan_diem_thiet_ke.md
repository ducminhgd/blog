# Kiến trúc: Quan điểm thiết kế

Đây là bài viết đầu tiên cho Series *Kiến trúc hệ thống và ứng dụng*, series này mình viết dựa trên kinh nghiệm và góc nhìn của cá nhân nên việc đúng hay sai thì không phân định rõ được, người đọc nên xem xét kỹ tình huống, vấn đề của chính mình mà quyết định.

Phạm vi bài viết:
- Chỉ nói về kiến trúc hệ thống và ứng dụng phần mềm, nên sẽ gọi tắt chung là *kiến trúc*
- *Ứng dụng* sẽ được dùng để gọi chung cho các ứng dụng frontend, ứng dụng backend (hay còn gọi là các dịch vụ - service).

Về quan điểm thiết kế thì mình chỉ có hai quan điểm chính là:
- Phải có tính mở rộng.
- Phải đúng chuẩn.

Ứng dụng mẫu dùng để ví dụ: một ứng dụng xem sách trực tuyến.

## Tính mở rộng

Tính mở rộng thể hiện ở một hệ thống hay ứng dụng có dễ dàng phát triển thêm module mới hay không, dễ dàng tích hợp hay không và dễ dàng thay thế hay không? Thay thế ở đây bao gồm cả nâng cấp và loại bỏ.

### Chia nhỏ

Trong nội bộ một hệ thống hay ứng dụng, nên phân rõ vai trò của một module và độc lập hóa vai trò đó. Nghĩa là một module chỉ làm một nhiệm vụ nhất định (Single Resposiblity trong SOLID), tránh việc module này phải làm việc của một module khác. Ví dụ: module quản lý sách phải độc lập với module quản lý người dùng. Vậy nếu muốn lấy danh sách sách của người dùng thì phải làm sao? Câu trả lời mà mình nghĩ OK nhất và mình hay làm nhất là tạo ra một module khác để nói lên mối quan hệ đó.

Vừa rồi mình nói là vai trò dựa trên đối tượng/thực thể (entity), và nếu theo một số ý từ trong Clean Architecture và theo một số design pattern như MVC, 3-layers,... thì mình nên phân rõ vai trò theo nhiệm vụ của module đó. Ví dụ: module lấy dữ liệu thì được gọi là Repository trong Clean Architecture, hay là Model trong MVC, hay là DAO trong 3-layers. Nếu để ý thì 3 cái cơ bản mà trong đa số thiết kế đều có là: thao tác dữ liệu (Data), xử lý nghiệp vụ (Business logic) và cuối cùng là giao tiếp tích hợp (Integration). 

### Tích hợp

Integration ở đây là việc đưa ra các interface để cho client hoặc các ứng dụng khác thực hiện các yêu cầu, việc trình duyệt gọi một đường dẫn trang web nào đó cũng là việc thực hiện yêu cầu, có thể coi là "tích hợp" giữa ứng dụng trình duyệt và ứng dụng.

Đối với một số ứng dụng, việc tích hợp ở đây còn bao gồm cả việc Authentication và Authorization cấp độ thấp. Ví dụ trong ứng dụng xem sách trực tuyến, có một dịch vụ quản lý thông tin sách *BookService*, dịch vụ này được hai dịch vụ khác là trang quản trị *Admin site* và dịch vụ cho thuê sách *RentService* gọi đến, *BookService* nên cung cấp một `access_id` và một `access_secret` để biết được những lời gọi là từ dịch vụ nào, đồng thời, cũng mở ra thêm được cho đối tác khác gọi nếu nhu cầu về nghiệp vụ/kinh doanh có thêm 3rd party cho thuê sách hoặc cần tra cứu thông tin sách. Tất nhiên, nếu mở rộng ra hơn nữa thì cần có một service quản lý Authentication và Authorization riêng, ví dụ: Identity and Access Management (IAM).

### Thay thế và loại bỏ

Thay thế ở đây là bao gồm cả việc nâng cấp và chuyển đổi từ một module này sang một module khác.

Loại bỏ ở đây là việc loại bỏ function, module, tính năng... mà không sử dụng nữa.

Nếu là việc thay để để nâng cấp thì phải đáp ứng được tính tương thích, ví dụ lên phiên bản mới mà các bên gửi và nhận yêu cầu chưa cập nhật kịp thì cũng không bị ảnh hưởng. Ngoài ra, việc thay thế để chuyển module thì bỏ một module này cũng không ảnh hưởng nhiều đến logic hay các module khác.

Phương pháp cho việc thay thế hay được áp dụng là: versioning, A/B testing, Blue/Green deployment.

Mục tiêu cuối cùng của việc dễ dàng thay thế và loại bỏ là đảm bảo tính tương thích *Compatibility*.

## Đúng chuẩn

Mình từng có nói về vụ 3 cấp độ của làm việc (Make it) là: *make it work*, *make it right* và *make it better*. Mục tiêu của mình ngay từ đầu luôn là *make it right* rồi chuyển dần lên *make it better*, nghĩa là cứ làm đúng trước đã, nếu sai thì còn các cái để so sánh mà sửa lại.

*Chuẩn* ở đây là tập hợp các bộ quy tắc, các đề xuất, các ràng buộc mà được phần đông người đồng ý; nó không nhất thiết phải đúng, chỉ cần có đông người xài. Mục đích là để đáp ứng quan điểm *có tính mở rộng* dễ dàng, giao tiếp dễ. Cứ thử tưởng tượng bạn nhờ một người khác nêm nếm lại nồi phở bạn đang nấu mà người ta không biết phở nấu thế nào hoặc không biết bạn đã nấu thế nào thì làm sao mà nêm tốt được (mặc dù có vài chuyên gia thì làm sao cũng tốt).

## Kết

Làm Software Development, thì phải làm sao cho cái Software của mình Developable được và có thể có người thay mình, phụ mình để Develop được. Và gần nhất, cụ thể nhất đó là *mình của tương lai* phát triển và vận hành cái phần mà *mình của quá khứ* và *mình của hiện tại* làm. Nếu làm một thứ mà mình còn không rõ ràng, không biết nó thế nào thì mình chỉ là làm rối lên thôi, làm bậy cho chạy.