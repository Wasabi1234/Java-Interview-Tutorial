这天风和日丽，小a正在工位上苦练钓鱼技术，
![](https://img-blog.csdnimg.cn/20210413101116274.png)

突然接到产品的☎️，又来需求？
![](https://img-blog.csdnimg.cn/20210413101305688.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)
只听到产品又开始口若黄河：我需要要查询到city是“上海”的所有人的name，并且还要按name排序返回前1000人的name、age。

 小a急忙正襟危坐，从一堆库表中翻出需要的表，抽出其建表语句：
![](https://img-blog.csdnimg.cn/20210413124053932.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)

看看表结构，再看看产品的需求
![](https://img-blog.csdnimg.cn/20210413134110379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)



感觉很容易，随手SQL这么一写：
![](https://img-blog.csdnimg.cn/20210413105428402.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)
诶，这语句看着简单而朴实，一个需求好像就完美解决了。但为了显示自己强大的性能优化水平，考虑到要避免全表扫描，于是又给 **city** 字段加索引。
建完索引，自然还需要使用explain验证一下：
```java
 explain select city, name, age from citizen where city = '上海' order by name limit 1000;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | citizen | NULL       | ALL  | city          | NULL | NULL    | NULL |   32 |   100.00 | Using where; Using filesort |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```

Extra字段的 **Using filesort** 表示需要排序，MySQL会给每个线程分配一块内存用于排序，称为**sort_buffer**。

这时魔鬼产品突然凑过来问：给我看看你代码咋写的，你这么写你真的懂MySQL 底层怎么执行order by的吗？
小a突然惊醒，还真没想过这些。

产品经理冷笑道：你知道你的 city 索引长啥样吗？
我自己建立的，我咋可能不知道！随手直接画出
- city字段的索引示意图
![](https://img-blog.csdnimg.cn/20210406195738724.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)

产品，你可好好看了，这里 `id_x ~ id_(x+n)` 的数据都满足city='上海’。

产品：那你倒是说说这条SQL的执行流程？不知道了吧，我来告诉你吧：
1. 初始化**sort_buffer**，确定放入`name、city、age`三字段
2. 从索引`city`找到第一个满足`city='上海’`条件的主键id， 即`id_x`；
3. 到id主键索引取出整行，取`name、city、age`三个字段的值，存入**sort_buffer**
4. 从索引city取下一个记录的主键id
5. 重复3、4，直到city的值不满足查询条件，即主键`id_y`
6. 对**sort_buffer**中数据按`name`做快排
7. 取排序后结果的前1000行返回给客户端

这就是**全字段排序**，执行流程如下：
![](https://img-blog.csdnimg.cn/20210412145927471.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)

**按name排序** 这一操作可能在内存中完成，也可能需要外部排序，而这就取决于
- 排序所需内存
- 参数**sort_buffer_size**
MySQL为排序开辟的内存（sort_buffer）的大小。若要排序的数据量小于**sort_buffer_size**，排序就在内存中完成。若排序数据量太大，内存放不下，则得利用磁盘临时文件辅助排序。

产品又开始炫技了，又问到：你知道 `一条排序语句何时才会使用临时文件` 吗？
这？这还真又触及到我的知识盲区了！
![](https://img-blog.csdnimg.cn/20210413134237949.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)
```java
mysql> SET optimizer_trace='enabled=on';
Query OK, 0 rows affected (0.00 sec)

/* 使用 @a 保存 Innodb_rows_read 的初始值 */
mysql> select VARIABLE_VALUE into @a from  performance_schema.session_status where variable_name = 'Innodb_rows_read';
Query OK, 1 row affected (0.00 sec)

mysql> select city, name,age from citizen where city='上海' order by name limit 1000;
+--------+------+-----+
| city   | name | age |
+--------+------+-----+
| 上海   | java |  22 |
...

/* 查看 OPTIMIZER_TRACE 输出 */
SELECT * FROM `information_schema`.`OPTIMIZER_TRACE`\G

/* 使用 @b 保存 Innodb_rows_read 的当前值 */
mysql> select VARIABLE_VALUE into @b from performance_schema.session_status where variable_name = 'Innodb_rows_read';
Query OK, 1 row affected (0.00 sec)

/* 计算 Innodb_rows_read 的差值 */
mysql> select @b-@a;
```
查看 **OPTIMIZER_TRACE** 结果中的 **number_of_tmp_files** 字段确认是否使用临时文件。

```sql
"filesort_execution": [
],
"filesort_summary": {
	"rows": 4000
	"examined_rows": 4000,
	"number_of_tmp_files": 12,
	"sort_buffer_size": 32664 ,
	"sort_mode": "<sort_key, packed_additional_fields>"
```
- **number_of_tmp_files**
排序过程中使用的临时文件数。为啥需要12个文件？内存放不下时，就需要使用外部排序，外部排序一般使用归并排序。
MySQL将需要排序的数据分成12份，每一份单独排序后存在这些临时文件中。然后把这12个有序文件再合并成一个有序的大文件。

若 **sort_buffer_size** 超过需排序的数据量大小，则 **number_of_tmp_files** 就是0，即排序可直接在内存完成。

否则就需要放在临时文件中排序。**sort_buffer_size**越小，需要分成的份数越多，**number_of_tmp_files**的值就越大。
- **examined_rows**
参与排序的行数。测试表有4000条满足city='上海’的记录，所以该参数为4000。

- **sort_mode** 的**packed_additional_fields**
排序过程对字符串做了“紧凑”处理。即使name字段的定义是varchar(16)，在排序过程中还是要按实际长度分配空间。

`select @b-@a` 的结果4000，即`整个执行过程只扫描了`4000行。

注意，为了避免对结论造成干扰，我把**internal_tmp_disk_storage_engine**设置成MyISAM。否则，`select @b-@a`的结果会显示为4001。
因为查询**OPTIMIZER_TRACE**表时，需要用到临时表，而**internal_tmp_disk_storage_engine**的默认值是InnoDB。若使用InnoDB，把数据从临时表取出时，会让**Innodb_rows_read**的值加1。

我惊奇地望着产品，像瞻仰伟人一般，不如你继承我的代码吧，让我来做产品？
![](https://img-blog.csdnimg.cn/20210413142731670.png)
`rowid排序`
上面的算法，只是对原表数据读了一遍，剩下的操作都是在**sort_buffer**和临时文件中执行。但这就存在问题：若查询要返回的字段很多，那么**sort_buffer**要放的字段数就会很多，内存里能够同时放下的行数就会变少，就要分成很多临时文件，排序性能就会很差。
**所以若单行很大，该方法的效率可不够行哦。**
![](https://img-blog.csdnimg.cn/20210413143147457.png)
产品大大又开始发难，那么你知道若**MySQL认为排序的单行长度太大，它又会干啥吗**？

现在修改个参数，让MySQL采用另外一种算法。
```bash
SET max_length_for_sort_data = 16;
```
- **max_length_for_sort_data**
MySQL用于控制**用于排序的行数据的长度**。若单行的长度超过该值，MySQL就认为单行太大，要换个算法。

`city、name、age` 三字段的定义总长度36，那你看我把**max_length_for_sort_data**设为16会咋样。

新的算法放入**sort_buffer**的字段，只有要排序的列（即name字段）和主键id。
但这时，排序的结果就因少了`city`和`age`字段值，不能直接返回了，整个执行流程变成如下：
1. 初始化sort_buffer，确定放入两个字段，即`name`和`id`
2. 从`city`找到第一个满足city='上海’条件的主键id，也就是图中的id_x
3. 到id取出整行，取name、id这两个字段，存入sort_buffer
4. 从`city`取下一个记录的主键id
5. 重复步骤3、4直到不满足city='上海’，也就是图中的id_y
6. 对sort_buffer中的数据按照字段name进行排序
7. 遍历排序结果，取前1000行，并按照id的值回到原表中取出city、name和age三个字段返回给客户端。

![](https://img-blog.csdnimg.cn/20210413144202883.png)
听到这里，感觉明白了一些：产品你别急，你看我画下这个`rowid排序`执行过程的示意图，看看对不对？
![](https://img-blog.csdnimg.cn/20210412150358621.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)
你看这个和你之前画的全字段排序示意图，其实就是多访问了一次表citizen的主键索引，即step7。

> 注意了，最后的resultSet是一个逻辑概念，实际上MySQL服务端从排序后的**sort_buffer**中依次取出id，然后到原表查到city、name和age这三字段的结果，不需要在服务端再耗费内存存储结果，而是直接返回给客户端。

这时查看rowid排序的**OPTIMIZER_TRACE**结果，看看和之前的不同之处在哪里

```sql
"filesort_execution": [
],
"filesort_summary": {
	"rows": 4000
	"examined_rows": 4000,
	"number_of_tmp_files": 10,
	"sort_buffer_size": 32728 ,
	"sort_mode": "<sort_key, rowid>"
```
- `select @b-@a`结果变成5000
因为这时除了排序过程，在排序完成后，还要根据id去原表取值。由于语句是limit 1000，因此会多读1000行。
- **sort_mode** 变成了 **<sort_key, rowid>**
表示参与排序的只有name和id字段
- **number_of_tmp_files** 变成10
因为这时参与排序的行数虽然还是4000，但每行都变小了，因此需排序的总数据量就小了，需要的临时文件也就少咯。

产品最后总结到：
- 若MySQL认为排序内存太小，会影响排序效率，就会采用rowid排序
这样排序过程中一次可以排序更多行，但最后需要回表取数据
- 若MySQL认为内存够大，会优先选择全字段排序
把需要字段都放到sort_buffer，这样排序后就直接从内存返回查询结果，不用回表。

所以MySQL就是：若内存够，就多利用内存，尽量减少磁盘访问。

对InnoDB，**rowid排序会要求回表，多造成了磁盘读，因此不会被优先选择**。
所以MySQL排序是个高成本操作。
- 是不是所有order by都需排序呢？若不排序就能得到正确的结果，那对系统的消耗会小很多，语句的执行时间也会变得更短。

并非所有order by都需排序操作。MySQL之所以需要生成临时表，并且在临时表上做排序，是因为原来的数据都是无序的。

- 如果能保证从city索引上取出来的行，天生就是按name递增排序，是不是就可以不用再排序了？
是的。

所以可以创建一个`city,name`联合索引：
```sql
alter table t add index citizen(city, name);
```

- 该索引的示意图
![](https://img-blog.csdnimg.cn/202104111826435.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)
依然可以用树搜索定位到第一个满足city='上海’的记录，并且能确保接下来按顺序取“下一条记录”的遍历过程，只要city是上海，name值一定有序。
这样整个查询过程的流程就变成：
1. 从索引(city,name)找到第一个满足city='上海’条件的主键id
2. 到主键id索引取出整行，取name、city、age三个字段的值，作为结果集的一部分直接返回
3. 从索引(city,name)取下一个记录主键id
4. 重复步骤2、3，直到查到第1000条记录，或者是不满足city='上海’条件时循环结束

- 引入(city,name)联合索引后，查询语句的执行计划
![](https://img-blog.csdnimg.cn/20210412151401991.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)

可见，该查询过程无需临时表，也无需排序。

- 使用 explain 查看(city,name)联合索引，查询语句的执行计划

```sql
 explain select city, name, age from citizen where city = '上海' order by name limit 1000;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | citizen | NULL       | ref  | city,name       | name | 51    | const |   4000 |   100.00 | Using index condition |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```
可见Extra字段中没有Using filesort了，也就是不需要排序了。而且由于(city,name)这个联合索引本身有序，所以该查询也不用把4000行全都读一遍，只要找到满足条件的前1000条记录即可退出。在这个例子里，只需扫描1000次。

该语句的执行流程有没有可能进一步简化呢？
- 覆盖索引
索引上的信息足够满足查询请求，不需要再回到主键索引上去取数据。

按覆盖索引，可以再优化一下这个查询语句的执行流程。
针对这个查询，我们可以创建一个city、name和age的联合索引，对应的SQL语句就是：

```sql
alter table t add index city_user_age(city, name, age);
```

这时，对于city字段的值相同的行来说，还是按照name字段的值递增排序的，此时的查询语句也就不再需要排序了。这样整个查询语句的执行流程就变成了：
1. 从索引(city,name,age)找到第一个满足city='上海’条件的记录，取出其中的city、name和age这三个字段的值，作为结果集的一部分直接返回
2. 从索引(city,name,age)取下一个记录，同样取出这三个字段的值，作为结果集的一部分直接返回
3. 重复2，直到查到第1000条记录或不满足city='上海'

引入 `(city,name,age)` 联合索引，查询语句的执行流程
![](https://img-blog.csdnimg.cn/20210412151620798.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_SmF2YUVkZ2U=,size_16,color_FFFFFF,t_70)
- explain查看`(city,name,age)`联合索引查询语句的执行计划

```sql
explain select city, name, age from citizen where city = '上海' order by name limit 1000;
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
| id | select_type | table   | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra                       |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
|  1 | SIMPLE      | citizen | NULL       | ref  | city,name,age        | age | 51    | const |   4000 |   100.00 | Using where; Using index |
+----+-------------+---------+------------+------+---------------+------+---------+------+------+----------+-----------------------------+
1 row in set, 1 warning (0.00 sec)
```
Extra字段里面多了“Using index”，说明使用了覆盖索引，性能上会快很多。
但这并非说要为了每个查询能用上覆盖索引，就要把语句中涉及的字段都建上联合索引，毕竟索引也很占空间，而且修改新增都会导致索引改变，还是具体业务场景具体分析。

> 参考
> - “order by”是怎么工作的？
> - https://blog.csdn.net/Linuxhus/article/details/112672434?utm_medium=distribute.pc_relevant.none-task-blog-baidujs_title-0&spm=1001.2101.3001.4242
> - 天天写order by，你知道Mysql底层执行原理吗？