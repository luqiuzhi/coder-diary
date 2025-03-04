# PostgreSQL中的索引长度

```text
### Error updating database.  Cause: org.postgresql.util.PSQLException: ERROR: index row size 3080 exceeds btree version 4 maximum 2704 for index "sys_interface_log_query_index"
  Detail: Index row references tuple (16112,6) in relation "sys_interface_log".
  Hint: Values larger than 1/3 of a buffer page cannot be indexed.
Consider a function index of an MD5 hash of the value, or use full text indexing.
### The error may exist in com/qk/framework/log/mapper/SysInterfaceLogMapper.java (best guess)
### The error may involve com.qk.framework.log.mapper.SysInterfaceLogMapper.insert-Inline
### The error occurred while setting parameters
### SQL: INSERT INTO sys_interface_log  ( request_id, type, business_key, interface_url, origin_payload, origin_response, business_data, result, request_time, external_system )  VALUES  ( ?, ?, ?, ?, ?, ?, ?, ?, ?, ? )
### Cause: org.postgresql.util.PSQLException: ERROR: index row size 3080 exceeds btree version 4 maximum 2704 for index "sys_interface_log_query_index"
  Detail: Index row references tuple (16112,6) in relation "sys_interface_log".
  Hint: Values larger than 1/3 of a buffer page cannot be indexed.
Consider a function index of an MD5 hash of the value, or use full text indexing.
; ERROR: index row size 3080 exceeds btree version 4 maximum 2704 for index "sys_interface_log_query_index"
  Detail: Index row references tuple (16112,6) in relation "sys_interface_log".
  Hint: Values larger than 1/3 of a buffer page cannot be indexed.
Consider a function index of an MD5 hash of the value, or use full text indexing.; nested exception is org.postgresql.util.PSQLException: ERROR: index row size 3080 exceeds btree version 4 maximum 2704 for index "sys_interface_log_query_index"
  Detail: Index row references tuple (16112,6) in relation "sys_interface_log".
  Hint: Values larger than 1/3 of a buffer page cannot be indexed.
Consider a function index of an MD5 hash of the value, or use full text indexing.
```