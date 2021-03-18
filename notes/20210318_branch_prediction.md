# Branch Predition

Phần lớn các kiến trúc CPU hiện đại bây giờ đều sử dụng Pipeline để chạy từng lệnh một trong một tập lệnh. Khi tập lệnh có điều kiện rẽ nhánh thì ta sẽ không biết trước được kết quả chạy của hiện tại là gì, khi đó hoặc là đợi hoặc là dự đoán. Branch Prediction là dùng để dự đoán kết quả chính xác hơn để chuẩn bị chạy lệnh tiếp theo, nếu dự đoán sai thì chạy lại.

Branch Prediction giúp tăng tốc độ của clock trong clock cycle và Instruction Level Paralellism (ILP - tạm dịch: chạy lệnh song song).

Tham khảo:
- https://en.wikipedia.org/wiki/Branch_predictor
- https://vnhacker.blogspot.com/2018/09/khai-tran-cong-nghe-chip-may-tinh-tien.html