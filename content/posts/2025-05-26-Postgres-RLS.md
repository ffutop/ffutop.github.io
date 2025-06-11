---
title: Postgres 行级安全策略
author: ffutop
date: 2025-05-26
categories:
  - 技术
tags:
  - Postgres
  - RLS
---

之前一直以为行级安全策略(RLS, Row-Level Security)是由 Supabase 自主开发的扩展特性，没成想它竟是由 Postgres 9.5 引入的官方特性。

由于一些需求，最近花了些时间来学习 RLS 的使用，在此记录一下。

## 1 RLS 能做什么

在 RLS 出现之前，如果想要在数据库中实现行级安全策略，通常有以下两种方式：

- 在应用代码中添加自定义过滤逻辑，例如在查询中添加 WHERE 条件，例如：
  ```sql
  SELECT * FROM users WHERE tenant_id = current_tenant();
  ```
- 创建视图，根据不同的访问模式来提供不同的查询视图，例如：
  ```sql
  CREATE VIEW users_view AS SELECT * FROM users WHERE tenant_id = current_tenant();
  ```
但是，这两种方式都存在一些问题：
- 应用代码中的过滤逻辑容易被遗忘，导致安全漏洞
- 创建视图会导致查询变得复杂，特别是当访问模式变得复杂时

RLS 通过将应用层的安全策略下沉到数据库级别，提供一种开箱即用的方案。

## 2 创建 RLS 策略

RLS 策略是一个函数，它会在查询数据之前被调用，根据当前用户的权限来过滤数据。
下面是一个简单的 RLS 策略：
```sql
CREATE OR REPLACE FUNCTION rls.users_policy()
RETURNS policy AS $$
BEGIN
  RETURN 'SELECT, INSERT, UPDATE, DELETE ON users WHERE tenant_id = current_tenant()';
END;
$$ LANGUAGE plpgsql;

ALTER TABLE users ENABLE ROW LEVEL SECURITY;
CREATE POLICY users_policy ON users FOR ALL USING (rls.users_policy());


當資料列安全原則在資料表裡被啓動後（使用 ALTER TABLE ... ENABLE ROW LEVEL SECURITY），所有資料表的操作，就必須符合資料列安全原則的設定。（當然，資料表的擁有者並不受限於資料列安全原則。）如果資料表中未設定任何原則，那麼預設就是拒絕存取，意思就是任何資料列都不能被看見或修改。但如果是整個資料表的操作行為，像是 TRUNCATE 或 REFERENCES，就不會受到影響。

## 3 RLS 实现原理

## 4 总结
