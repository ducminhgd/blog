# Kinh nghiệm về ERP

Sau nhiều lần làm về ERP và gần đây gặp lại nhiều vấn đề của ERP nên mình muốn ghi chú lại để cho chính bản thân mình hoặc cho những ai muốn tham khảo.

## Lầm tưởng ERP là phần mềm kế toán, phần mềm nhân sự

ERP là Enterprise Resource Planning, là phầm mềm để quản trị nguồn lực của doanh nghiệp, từ đầu vào đến đầu ra; ERP là phần mềm quản trị tích hợp toàn bộ các phòng ban, bộ phận trong doanh nghiệp.

Phần mềm kế toán hay phần mềm nhân sự chỉ là một phần mềm chuyên môn hóa cho một phòng ban, chuyên sâu hóa cho các nghiệp vụ cụ thể nào đó. Và thường trong ERP, có bao gồm các phân hệ kế toán, nhân sự, đào tạo, CRM,...

## Lầm tưởng nhận định về khách hàng, người dùng cuối

Nhiều lần phát triển, triển khai ERP thì mình thấy có những quan điểm là:
- "Chị kế toán chính là người dùng, họ có kinh nghiệm, họ trực tiếp làm việc, cứ nghe theo họ"
- "Anh CEO mới là người trả tiền cho chúng ta, làm theo ảnh"

Và theo mình là, ai cũng chưa đủ. **ERP là phần mềm dành cho doanh nghiệp, nên vấn đề cần giải quyết là vấn đề của doanh nghiệp**, không phải của anh CEO hay chị kế toán, mà họ chỉ những nhóm trực tiếp liên quan đến các *vấn đề bề nổi* của doanh nghiệp. Vì sao là *bể nổi*? Vì họ khó chịu cho chính cái họ đang làm, họ muốn cái này, họ muốn cái kia, nhưng cái họ cần là gì thì người làm phần mềm của chúng ta phải tư vấn.

Để áp dụng ERP, có khi công ty phải thay đổi quy trình, phải thay đổi cơ cấu cho phù hợp để vận hành lâu dài và bền vững. Chứ làm theo anh CEO, chị kế toán, rồi mai mốt người khác thay họ thì mình lại phải thay đổi theo?

Triển khai một phần mềm ERP nó rất tốn tiền, nên cái mục đích của khách hàng (doanh nghiệp) là để giải quyết vấn đề của họ, bằng cách thống nhất quy trình, bằng cách minh bạch số liệu, bằng các bảng biểu mà họ nhìn vào có thể ra quyết định được,...

Người làm phần mềm cần làm gì? Hướng đến vấn đề của doanh nghiệp, từ những thứ bề nổi đi tìm những vấn đề sâu xa mà giải quyết, dung hòa "anh CEO", "chị kế toán", "cô bán hàng", "chú bảo vệ".

Còn việc lâu lâu "chiều tí" cho họ vui để họ vui vẻ hỗ trợ hoặc họ duy trì hợp đồng thì phần mềm nào cũng có, ERP cũng có. Mà phải có ranh giới rõ ràng, tránh tự "đè chân" mình về sau.

## Lầm tưởng về sức mạnh của công nghệ

Đây là lầm tưởng đến phần lớn từ mấy anh em Dev, kiểu như *quy trình, nghiệp vụ khó cỡ nào cũng thì mình code cũng được*. Ví dụ: một bạn C&B không biết mã hóa file phiếu lương, hoặc bạn hay gửi nhầm phiếu lương. Việc của mình là vừa đào tạo, vừa hỗ trợ, chứ không phải dùng công nghệ để làm giùm bạn. Bạn không biết thì bạn phải học, bạn ẩu thì bạn phải sửa. Chứ không phải vì một bạn ẩu mà tốn 30-40 ngày làm việc để làm tính năng mã hóa, giải mã, gửi phiếu lương. Việc làm đó gần như vô ích nếu không giải quyết vấn đề của doanh nghiệp, của nhân sự trong doanh nghiệp, nó chỉ có ích cho mấy ông làm phần mềm đi kiếm ăn thôi.

Mình vẫn còn nhớ câu của anh sếp cũ: "Đừng bao giờ dùng tin học đi giải quyết những vấn đề phi tin học".

Bản thân **ERP chỉ là một bộ công cụ** để giúp doanh nghiệp làm tốt hơn, không có phần mềm ERP không có nghĩa là doanh nghiệp chết. Bấy lâu nay họ vẫn vận hành được, họ đang muốn chuẩn hóa, và làm tốt hơn.

Thử một lần đi nói là "tính năng này làm được, tốn nhiều công sức, sợ hiệu quả mang lại chưa chắc cao, mình có thể sắp xếp làm sau không?", có khi nhận được những câu như là "ừ không sao, tiện thì làm, không cần lắm, cái này Excel/Công cụ khác làm được".

## Bỏ qua quy trình, quy định, pháp luật

Mấy ông Dev hay có cái kiểu là làm chạy được là được, hoặc bùa chú gì đó. Gần đây nhất, ở Vietnam Silicon, đổi benefit "nhẹ" thôi, là cho nhân viên nghỉ nửa ngày hưởng lương ngày 31/12, các anh em code cái phần nghỉ phép đó phải tốn một vài tuần để cover các case: xin nghỉ phép, block xin nghỉ chiều 31/12 (do được free), rồi cộng dồn phép (case này dính cộng dồn ngày hôm sau, cộng sau lễ, cộng qua năm),... Nhiều người ban đầu sẽ nghĩ là cộng cho họ 0.5 ngày, hoặc vào trừ của họ 0.5 ngày chiều 31/12.

Hồi xưa, mình đi team building của một tập đoàn, và họ tự động vào trừ 1 ngày phép năm của mình trên hệ thống. Mình nói thiệt là kệ, chứ họ làm vậy không đúng đâu, mình đâu có đồng ý. Họ bắt buộc mình đi team building, không được từ chối nếu không có lý do hợp lý, thì tính ra là ngày đó mình đang đi công tác cho họ, không phải mình nghỉ phép. Nêu cái vụ trừ 0.5 ngày phía trên, cũng có thể khiếu nại được.

Và sau đó thì cách giải quyết dung hòa nhất, hài lòng nhân viên, không tốn nhiều công sức là: một cái email dặn dò anh em là book phép liên tục thì chia ra làm nhiều phép, trong đó một cái sáng 31/12 thôi. Anh em ai cũng sẵn lòng làm, cái này cũng là một ví dụ để giải quyết cái "lầm tưởng về sức mạnh của công nghệ" nói trên.

## Lầm tưởng về kinh nghiệm

Một sự việc từng diễn ra khi mình làm ERP cho một doanh nghiệp bán lẻ lâu năm, khi làm đến phần kế toán, mình thấy là chị kế toán trưởng để cho mình làm việc với bạn trẻ hơn, rồi các bạn cũng có vẻ không hỗ trợ lắm, thế là mình phải dùng mối quan hệ cũ, đi hỏi thêm kiếm thức thì một chi kế toán trưởng và một bạn kế toán công ty cũ. Dần dần thì các bạn kế toán của công ty khách hàng hiện tại cũng cởi mở hơn, chị kế toán trưởng dần dần cũng chịu làm việc trực tiếp với mình.Lý do là gì: mình không có cùng ngôn ngữ với họ, mình chả biết gì về kế toán cả, sau khi mình học hỏi xong, thì họ thấy rằng mình hiểu được cái mà họ sẽ nói, nên họ bắt đầu chia sẻ nhiều hơn.

Đoạn trên chỉ là mở bài thôi, đoạn chính là đoạn này. Sau khi mình làm việc lâu dài với chị kế toán trưởng, chị ấy mới chia sẻ cho mình biết cái **quy trình của công ty là cái giúp cho công ty vận hành trơn tru, nó có chút điều chỉnh lại so với cái "quy trình chuẩn"** (chuẩn là theo pháp luật, nghị định, nghị quyết). Do cái "chuẩn" nó chưa rõ ràng, hoặc nó không linh hoạt, nên đa số công ty sẽ điều chỉnh lại cho phù hợp. Và, đối với những bạn quá trẻ, những bạn tay ngang, hoặc những bạn ban đầu chả biết gì thì về phía công ty, họ chỉ muốn đào tạo cho làm được việc trước đã, họ chẳng giải thích gì nhiều cả. Do đó, **cái kinh nghiệm mà người làm tại vị trí đó chưa chắc làm đúng cái họ đang làm**, họ chỉ đang làm theo cái được chỉ dẫn thôi.

Do đó, đừng nghĩ rằng một kế toán chắc chắn hiểu về kế toán. Đừng nghĩ một Sales giỏi lại đang làm đúng quy trình bán hàng. **Đừng nghĩ một người triển khai ERP lại hiểu ERP thật sự là gì**.

Câu cuối chính là câu kết bài, mình cũng đang nói về mình, những cái mình ghi chú ở đây, cũng chưa chắc là đúng, đó là kinh nghiệm của mình thôi.
