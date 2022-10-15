# Sibyl
# 基本介绍
## 团队名称：鸡你太美
- [lilyjazz](https://github.com/lilyjazz): TiDB 产品经理
- [Hawkson-jee](https://github.com/Hawkson-jee): TiDB 后端研发
- [lucklove](https://github.com/lucklove): TiDB 后端研发
- [shhdgit](https://github.com/shhdgit): TiDB 前端研发
## 项目进展
- 慢 SQL 和相关信息展示 DEMO 制作（ing）
- SQL Advisor DEMO 制作（ing）
## 项目介绍
Sibyl is an Easier-to-use SQL Diagnostics tool for TiDB. 小白也能用的 SQL 优化工具。
## 背景 & 动机
TiDB 作为一个分布式数据库管理系统，有着比单机数据库更复杂的 SQL 诊断难度。为了诊断一个慢查询，
- 需要理解不同 workload 适合什么样的存储类型（列存还是行存）？
- 需要深刻理解 TiDB 的执行计划（[TiDB 执行计划概览](https://docs.pingcap.com/zh/tidb/v6.3/explain-overview)），才能发现慢查询的问题，如：SQL 写的不对导致算子不合适，未能将执行计划成功下推到存储节点；哪些范围查询算子不能使用索引；哪些算子相关的系统变量设置有问题；等等。
[SQL Plan]()
- 需要熟悉 TiDB 的基本架构逻辑，理解不同组件耗时指标的正常阈值。当超出阈值时，需要用户清楚如何操作才能对异常值进行优化。
[SQL Exe Time]
[SQL Cop Time]
- 需要发现整体性能和单个 SQL 之间的关系。慢 SQL 数量的整体提升，可能是因为磁盘的抖动，或者单个 TiDB 的资源达到瓶颈？
[TiKV iops]
因此，虽然 TiDB 有丰富的文档介绍，包括 SQL 诊断的文档介绍，但是在 TiDB 的技术支持中，依然有客户寻求更多的慢查询的优化方案。
基于以上背景，为了让用户能够更顺利地使用 TiDB，需要能够将更多的 TIDB 支持的专家能力自动化的转化为产品对用户的建议，让用户能够自主了解问题，采用建议，独立完成慢查询的优化过程。
## 项目设计
### 整体思路
我们尝试优化 TiDB 的 Slow SQL 查询性能，提供多种的优化建议，包括不限于索引优化，表结构优化，配置和参数优化，拓扑优化等等。
[SQL Advisor Frame]
### Advisor Model Case
[Model Case Summary]
####  Case 1. 确少索引 和统计信息缺失
[Case1-1]
Model 输出：
1. Create index  idx_001 on orders(O_TOTALPRICE)
2. 建议用户执行analyse table orders， 收集统计信息。
[Case1-2]
#### Case 2: 有索引但是不走
[Case2-1]
[Case2-2]
输出： 解释原因 为什么不走，因为该条语句的结果集太大了，超过了80%。
#### Case 3. 如何判读一个语句是否是最优
[Case3-1]
如果返回结果集远远小于扫描行数 则需要优化。主要是索引建议
explain analyze select count(*) from orders where O_TOTALPRICE>3000 and O_CLERK='Tom' and O_ORDERDATE>'2022-09-30';
#### Case 4. 复合字段索引建议
Select * from orders where O_ORDERSTATUS ='F' and  O_ORDERPRIORITY='5-LOW' and O_SHIPPRIORITY!=0
[Case4-1]
分别计算各列的distinct值：
[Case4-2]
[Case4-3]
[Case4-4]
建复合索引的后台逻辑：
按照where条件中distinct值由到底的顺序：
1. O_ORDERPRIORITY，
2. O_ORDERSTATUS
3. O_SHIPPRIORITY
Create index idx_003 on orders (O_ORDERPRIORITY, O_ORDERSTATUS, O_SHIPPRIORITY)
[Case4-5]
[Case4-6]
时间提升计算逻辑： 以前实际扫描行数/加索引后扫描行数。这个这是一个近似值。但上面这个例子中如果按照这个算法可以提升80倍。但是实践提升只有5倍。之所以有这么大的差距主要是TiDB是分布式数据。和MySQL那种单体数据库内部执行逻辑不一样。
#### Case 5. 多字段查询，只有部分字段命中索引
如下列子：
Select * from orders where O_ORDERSTATUS ='F' and  O_ORDERPRIORITY='5-LOW' and O_Clerk='Clerk#000006445';
[Case5-1]
通过查询发现O_Clerk的distinc值有大概59000个，所以建议加下列索引：
Create table idx_004 on orders(O_Clerk,O_ORDERPRIORITY, O_ORDERSTATUS)
[Case5-2]
提升两倍。
#### Case 6. 统计信息过时
TODO
#### 索引字段的选择性计算逻辑
- 通过 “show table status like” 获得表的总行数 table_count。
- 通过计算选择表中已存在的区分度最高的索引 best_index，同时Primary key > Unique key > 一般索引。
- 通过计算获取数据采样的起始值offset与采样范围rand_rows：
  - offset = (table_count / 2) > 10W ? 10W : (table_count / 2)
  - rand_rows =(table_count / 2) > 1W ? 1W : (table_count / 2)
- 使用select count(1) from (select field from table force index(best_index) order by cl.. desc limit rand_rows) where field_print 得到满足条件的rows。
  - cardinality = rows == 0 ? rand_rows : rand_rows / rows;
## 缺点
### Automatic Running
自动化控制集群的功能，虽然可以在 Hackathon 中进行尝试，但是在真实业务集群中，系统跳过集群管理员直接操作集群的行为，还有一段时间要走。因为数据库集群的稳定对上层业务非常重要，所以在真实业务集群中会考虑提供建议操作，让用户来执行。
### Advisor Model 的反馈、评价和优化
虽然有很多专家经验可以参考，建立相对合理的建议模型。但是，依然不能排除有非常特殊的 Workload，让模型给出无效的优化操作。因此，后续需要建立对所有 Advisor 的反馈、评价和优化机制。
在后续的产品化工作中，需要了解每个 Advisor 被生成、被应用的次数，以及每一次应用后的优化效果，并且根据不同 workload 中的优化效果，来尝试对模型进行后续的优化。
## FAQ
TODO
## 效果验证
### 环境
TiDB v6.3
AWS EC2
### Workload
TBD
### 结果
TODO
