# Debezium connector

Hôm nay có dịp đụng đến Debezium Connector và có một số ý để ghi chú như sau

## Xóa connector, tạo lại, vẫn không có message?

Kafka là theo hướng "append only", nghĩa là chỉ thêm vào, những cái đã có, đã qua không được cập nhật. Do đó, nếu muốn chạy lại hoặc tạo lại một connector cùng tên thì phải vào Topics, kiếm topic tên `debezium-offset` và xóa các message có liên quan đến connector cũ.

`debezium-offset` là dùng để lưu trữ offset đã được lấy của một connector. Do đó, việc xóa message tương ứng với một cách reset lại offset.

Một cách khác là bắn thêm một message có cùng key nhưng có giá trị là `null`.

## Transform

https://docs.confluent.io/platform/current/connect/transforms/regexrouter.html#remove-part-of-a-topic-name


```json
{
    // ...
    "transforms": "RemoveString",
    "transforms.RemoveString.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.RemoveString.regex": "<database.server.name>.<database-name>.(.*)",
    "transforms.RemoveString.replacement": "<prefix>.$1.<suffix>",
    // ...
}
```

Trong đó, `<database.server.name>` là giá trị của field `"database.server.name"` trong config, `<database-name>` là tên của database mà source connector đang trỏ đến.

Trong trường hợp `transforms.RemoveString.replacement` là một giá trị cố định thì nhiều bảng sẽ được gửi vào cùng một topic.
