## Order By 如何工作？
如：`select id,name,age from t wher age>20 and age<50;` `age`上有索引，`id`是主键，则，语句执行会利用`age`
索引找到合适记录的主键`id`,然后会表查出 `name` 和 `age` 字段放入`sort buffer` 排序，这种称为全字段排序，若查询的字段信息特别大
，超过了 `max_length_for_sort_data`，那么只把`age,id`放入`sort buffer`排序，然后在会主键表查询返回客户端

故内存允许，MySQL优先使用`全字段排序`。优化措施使用覆盖索引，避免回表会排序
体现在explain 中就是 `using index condition（需要回表）,using file sort` 变为`using index`（使用覆盖索引）


## MySQL LRU 如何实现,一个全表扫描是否会导致Buffer缓存全部失效？
MySQL 的buffer一般建议设置到机器内存的60%-80%，如果正常的LRU 算法，大表的全表扫描将导致热点数据都被淘汰，
所以MySQL 的 LRU算法做了一个改进，将内存5:3划分yong区和old区.
* 数据第一次读入内存在old区，若在1s 内重复访问，不会移动到队头
* 若old区的数据在1s内重复访问，则移动到队头，改时间具体由参数`innodb_old_blocks_time`控制
* 若内存不足，直接移除队尾数据，也即old 队尾
* 若数据在yong区重新访问，则直接移到队头

备注：`show engine innodb status 可以查看内存命中率`,字段`Buffer pool hit rate`，一般命中率在99%

## Join 原理
假设 t1，t2 表结构一样，t2 有1000行数据（1,1,1）~（1000,1000,1000）而 t1 只有100行
（1,1,1）~（100,100,100）
```SQL
CREATE TABLE `t2` 
( `id` int(11) NOT NULL, 
`a` int(11) DEFAULT NULL, 
`b` int(11) DEFAULT NULL, 
PRIMARY KEY (`id`), KEY `a` (`a`)) ENGINE=InnoDB;

create table t1 like t2;
```

### Index Nested-Loop Join(NLJ)
则以下的查询
```SQL
select * from t1 straight_join t2 on (t1.a=t2.a);
```
t1 是驱动表，t2 是被驱动表，则全表扫描t1(100行)，每取出一行的`a`字段在 t1表中走索引查找，所以最终
t2 也只扫了100行，这个称为 `Index Nested-Loop Join（NLJ）`
假设t1小表行数M,t2大表行数N,小表驱动大表时，复杂度为：`M+MlogN`，当然前提是能使用大表的索引

### Block Nested-Loop Join(BNL)
```SQL
select * from t1 straight_join t2 on (t1.a=t2.b);//这种Block NLJ 不推荐使用
```
此时t2 没有使用索引  
join 流程：
1. 将t1 数据放入 `join buffer`
2.每次取一条t2数据，都和`join buffer` 数据做比较，也即总共比较了100*1000次
若t1 太大无法一次放入 `join buffer`,则分块放入，多次执行1~2步骤

若 `join_buffer_size`不够时，优先小表驱动大表，否则，效果一样

综上：不论是 `NLJ`还是 `BNL`,都要小表驱动大表，小表的的定义是：行数*查询的的字段大小

若小表不能一次放入`join buffer`,则需要分块放入，那么被驱动表需要多次扫描，若其大小小于LRU_old，
那么大表可能由于1s 内多次访问，被移动到LRU_yong，从而影响这个数据库的缓存命中率

### 优化BNL？
1. 大表上建立索引，将BNL转化为BKA
2. 查询频率不高，不想建索引，可以使用临时表
      
   如：
    `select * from t1 join t2 on (t1.b=t2.b) where t2.b>=1 and t2.b<=2000;`若使用BNL，
    则需要将t2表的全部数据和t1表联合后在进行判断where,计算量很大，可以使用临时表将t2表中`1<=b<=2000`
    数据放入，并建立索引，在和t1进行join,也即转为BKA

## Multi Rand Read (MRR)
```SQL
select * from t where a>100  ;
```
假设t的主键是id,a是索引，那么此时查询会使用索引a 查询数据，然后回表（使用主键索引）查询其他字段的值，
a 上有序，id 上可能乱序，那么MRR 会将id 按照升序的方式排序放在`read_rnd_buffer`，然后去读取数据页
这样能减少磁盘IO(顺序读优于随机读)

## Batched Key Access (BKA)
这个是用来优化NLJ，在NLJ中，每次从驱动表中取一行，然后去被驱动表中索引查找，使用BKA 就可以将驱动表中的多个行
放入 `join buffer` 来提升性能，不过这个前提是打开`MRR`

## 临时表和普通表的区别
1. 只能在同一个session 可见
2. 可以与普通表重名，不过优先使用临时表(临时表底层文件是数据库名+表面+线程号区分，所以还可以重名)
3. show tables;只能看见普通表
4. 临时表会写入磁盘（若使用内存引擎则不写入磁盘）
5. 连接断开，临时表自动被删除

备注：
1. 非常适用于Join优化，并发查询时，不同session创建的同名临时表不冲突
2. 临时表可以在分库分表是将多个分表的数据合并然后排序。
如`select v from ht where k >= M order by t_modified desc limit 100;`
每个分表都有k索引，需要现在每个分表按照索引查询后，在临时表中汇总排序（当然也可以业务侧或proxy排序）

主备同步
1. 若binlog format 是statement/mix ,则需要同步临时表。
否则：`insert into t_normal select * from temp_t;`将报错`temp_t`不存在，
由于主备同步线程是长期运行，会导致临时表长期有效，可以在主库drop temporary table temp_t,等到SQL 同步到备库也会删除备库数据，
若binlog format 是row 则不需要同步临时表


## MySQL什么时候会使用临时表
1. (select id from t1) union (select id from t2);由于union 会将重复的行过滤，会采用临时表判断，union all 则不会，因为其不会过滤相同行
2. select id%10 as i,count(*) from t group by i ; 会将`t`表的`id%10`存入临时表，然后使用`sort buffer`排序
,在按照i `group by`,在放入内存临时表，然后返回

注意：
1. explain 中会发现`Using temporary`
2. group by 语句的结果没有排序要求，要在语句后面加 order by null，可以避免排序

上面的`group by`如何优化？
1. 表中t加一个字段c 存 id%10 ,并对c 建立索引，则是不需要使用临时表还排序
2. `select SQL_BIG_RESULT id%100 as m, count(*) as c from t1 group by m;`直接告诉优化器，
这个数据量很大，直接放入磁盘，避免了临时表先放内存，放不下再转移到磁盘的过程

## MySQL 分区表
若一个表有四个分区，对应 一个`.frm` 和四个 `.ibd`, 在server 层是一个表，引擎层是4个表

不推荐使用分区表原因
1. 若分区很多，MySQL启动时会打开很多文件描述符，可能导致`Too many open files`
2. server 层所有表共用一个MDL锁，也即更新DDL将导致全部分区的操作阻塞
3. 若where 条件没有包含分区，将导致扫描所有的分区

替代方案： 手工建立多个普通表，在业务层进行路由到对应的表
分区表优势：`alter table t drop partition`快速删除历史数据；对业务透明

## 权限管理
1. 创建用户：`create user 'ua'@'%' identified by 'pa';`
   用户有用户名+ip组成，可以在`select * from mysql.user;`中查到用户
2. 全局权限：`grant all privileges on *.* to 'ua'@'%' with grant option;`
    对已存在的连接不受影响
3. DB 权限：`grant all privileges on db1.* to 'ua'@'%' with grant option;`
   可以在`select * from mysql.db ;`中查到权限
   若用户已经 `use test;`进入某个数据库，则后续的DB权限修改不影响他，除非他退出该DB
4. 表和列权限
    `grant all privileges on db1.t1 to 'ua'@'%' with grant option;GRANT SELECT(id), INSERT (id,a) ON mydb.mytbl TO 'ua'@'%' with grant option;`
    这个权限修改是即时生效

备注:正常不需要 `flush privileges`，除非某些异常导致数据库里面db的权限和内存不一致才需要使用