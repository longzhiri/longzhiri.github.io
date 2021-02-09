---
layout: post
title: MySQL导入导出指定列
tags: Database
---

一般导入数据是按照行导入，但是有时只想导入一行中特定字段，一种可行的方案如下：

以表 test_out(id, data) 和 test_in(name, data)为例，想导入test_out.data字段到test_in.data中。

导出时同时生成sql语句：
```sql
 select concat("update test_in set data=(X'", hex(data), "') where name='xxx';") into outfile "/tmp/import.sql" from test_out where id=1;
 ```
 把test_out中id=1的data字段导入到test_in中name='xxx'的行。
 data如果是二进制数据需要hex函数导出成16进制字符串，然后再以 X'...' 的方式导入16进制字符串。

 当然也可以先在要导入的数据库生成一个和test_out表一样的临时表tmp_test_out，全量导入指定行，然后再把tmp_test_out中的指定列update到test_in，最后删除tmp_test_out。
