---
title: Oracle insert into优化及测试
date: 2020-01-20 10:40:23
tags:
categories:
---
# Oracle insert into慢，优化方案选择和测试
Oracle端
- 默认插入耗时
- Nologing插入耗时
- APPEND 不寻址，不去寻找 freelist 中的free block , 直接在table HWM 上面加入数据
- parallel 并行
- 临时表+批量插入耗时
## 创建测试环境
1. 表结构：
```
-- Create table
create table LOG_OPERATING
(
  id            VARCHAR2(20) not null,
  type          CHAR(1) not null,
  operator      VARCHAR2(50) not null,
  operator_id   VARCHAR2(50) not null,
  url           VARCHAR2(255),
  ip            VARCHAR2(65) not null,
  model         VARCHAR2(40) not null,
  description   VARCHAR2(255) not null,
  time          DATE not null,
  is_successful CHAR(1) default 1
)
tablespace USERS
  pctfree 10
  initrans 1
  maxtrans 255
  storage
  (
    initial 64K
    next 1M
    minextents 1
    maxextents unlimited
  );
-- Create/Recreate primary, unique and foreign key constraints 
alter table LOG_OPERATING
  add constraint PK_LOG_OPERATING primary key (ID)
  using index 
  tablespace USERS
  pctfree 10
  initrans 2
  maxtrans 255
  storage
  (
    initial 64K
    next 1M
    minextents 1
    maxextents unlimited
  );
```
2. Oracle版本
```
SQL>select * from v$version;
BANNER
---------------------------------------------------------------------
Oracle Database 11g Enterprise Edition Release 11.2.0.1.0 - 64bit Production
PL/SQL Release 11.2.0.1.0 - Production
CORE 11.2.0.1.0 Production
TNS for Linux: Version 11.2.0.1.0 - Production
NLSRTL Version 11.2.0.1.0 - Production
```
## Insert测试
1. 默认操作，无任何优化
```
declare i int:=1;
begin
  while i<=1000000 loop
insert into log_operating
  (id,
   type,
   operator,
   operator_id,
   url,
   ip,
   model,
   description,
   time,
   is_successful)
values
  (i,
   '1',
   'pki_user',
   '399262438320640000',
   '/v1/common/dict/certApplyType',
   '192.168.1.1',
   '通用方法',
   '获取该类型下的值',
   SYSDATE,
   '1');
  i:=i+1;
  end loop;
end
commit;
```
耗时：77.298秒

2.添加Nologing
```
alter table log_operating nologging; 
--insert
alter table log_operating logging;
```
耗时：68.259秒

3.添加 /*+ append */
```
declare i int:=1;
begin
  while i<=1000000 loop
insert /*+ append */ into log_operating
  (id,
   type,
   operator,
   operator_id,
   url,
   ip,
   model,
   description,
   time,
   is_successful)
values
  (i,
   '1',
   'pki_user',
   '399262438320640000',
   '/v1/common/dict/certApplyType',
   '192.168.1.1',
   '通用方法',
   '获取该类型下的值',
   SYSDATE,
   '1');
  i:=i+1;
  end loop;
end
commit;
```
耗时：67.258秒

3.NOloging+/*+ append */
```
alter table log_operating nologging;
--插入
```
耗时：68.85秒

4.临时表+NOloging+/*+ append */
```
alter table log_operating nologging;


declare i int:=1;
begin
  while i<=1000000 loop
insert into TEMP_LOG_OPERATING
  (id,
   type,
   operator,
   operator_id,
   url,
   ip,
   model,
   description,
   time,
   is_successful)
values
  (i,
   '1',
   'pki_user',
   '399262438320640000',
   '/v1/common/dict/certApplyType',
   '192.168.1.1',
   '通用方法',
   '获取该类型下的值',
   SYSDATE,
   '1');
  i:=i+1;
  end loop;
  insert /*+ append */ into LOG_OPERATING select * from TEMP_LOG_OPERATING;
  truncate table TEMP_LOG_OPERATING;
end
commit;
```
耗时：54.377秒

5.临时表+NOloging+/*+ append */+parallel
```
alter table TEMP_LOG_OPERATING nologging;
alter table LOG_OPERATING nologging;

declare i int:=1;
begin
  while i<=1000000 loop
insert into TEMP_LOG_OPERATING
  (id,
   type,
   operator,
   operator_id,
   url,
   ip,
   model,
   description,
   time,
   is_successful)
values
  (i,
   '1',
   'pki_user',
   '399262438320640000',
   '/v1/common/dict/certApplyType',
   '192.168.1.1',
   '通用方法',
   '获取该类型下的值',
   SYSDATE,
   '1');
  i:=i+1;
  end loop;
  insert /*+ append parallel(a, 4) nologging */  into LOG_OPERATING select /*+ parallel(b, 4) */ * from TEMP_LOG_OPERATING;
end
commit;
 truncate table TEMP_LOG_OPERATING;
```
耗时：42.173

## 结论
以上结果可能存在误差，建议使用批量插入+临时表+APPEND+PARELLEL




参考文章：
https://blog.csdn.net/S630730701/article/details/71732405
https://blog.csdn.net/tmaczt/article/details/84134173
