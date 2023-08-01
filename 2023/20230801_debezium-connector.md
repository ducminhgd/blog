# Debezium connector

- Kafka là append only, do đó phải vào topic debezium-offset xóa đi message offset cho lấy lại từ đầu
- Transform

```json
{
    ...
    "transforms": "RemoveString",
    "transforms.RemoveString.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.RemoveString.regex": "<database.server.name>.<database-name>.(.*)",
    "transforms.RemoveString.replacement": "<prefix>.$1.<suffix>",
    ...
}
```