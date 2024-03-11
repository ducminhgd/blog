# Database Part 4: FQA

## Is database or data important? And why? We can handle them with our code

You SHOULD remember that the input of your codes are data, and the results are the data or the changes of data. When you have a good design data or database, the effort you spend to develop or maintain should be less.

Another reason is the performance of code sometimes relates to the data they process, or the database that they connect to. Most of the high latency cases are come from network problems or the bottle-neck problem of the database.

## Which data type should we use for `status`?

From my experiences, there are multiple otpions: `BOOLEAN`/`BIT`, `SMALLINT`/`TINYINT`, `INT`, Enums, Strings. And, I recommend to use the smallest type of integer. Because:

- If you choose binary data types, you cannot extend the values for that fields. And sometimes, you need more columns to describe the status, for example `is_draft`, `is_approved`, `is_published`.
- If you choose Enum or Strings, there is a ridiculous (but make sense) reason, typos make the application errors.
