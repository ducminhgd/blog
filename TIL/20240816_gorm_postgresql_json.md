# Gorm & PostgreSQL & JSON


```go
r.db.DB().Model(&models.Strategy{}).WithContext(ctx).
Where("feature_id = ?", f.ID).
Where(datatypes.JSONQuery("data").HasKey(conditions.Key)).
Where(datatypes.JSONQuery("data").HasKey(conditions.Key, conditions.Value)).
Count(&count)
```

```sql
SELECT count(*) FROM "strategy" WHERE feature_id =  '01915589-f09c-70bf-bac1-26e1a362eea3'  AND "data"::jsonb ?  'region'  AND "data"::jsonb ->  'region'  ?  'vn'
```