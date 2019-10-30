版权声明：本文为博主原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接和本声明。
本文链接：https://blog.csdn.net/vbirdbest/article/details/81065566
一：基础数据准备
```
DROP TABLE IF EXISTS `tbl_user`;
CREATE TABLE `tbl_user` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `username` varchar(255) DEFAULT NULL,
  `email` varchar(20) DEFAULT NULL,
  `age` tinyint(4) DEFAULT NULL,
  `type` int(11) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=10 DEFAULT CHARSET=utf8;

INSERT INTO `tbl_user` VALUES 
('1', 'admin', 'admin@126.com', '18', '1', '2018-07-09 11:08:57'), 
('2', 'mengday', 'mengday@163.com', '31', '2', '2018-07-09 11:09:00'), 
('3', 'mengdee', 'mengdee@163.com', '20', '2', '2018-07-09 11:09:04'), 
('4', 'root', 'root@163.com', '31', '1', '2018-07-09 14:36:19'), 
('5', 'zhangsan', 'zhangsan@126.com', '20', '1', '2018-07-09 14:37:28'), 
('6', 'lisi', 'lisi@gmail.com', '20', '1', '2018-07-09 14:37:31'), 
('7', 'wangwu', 'wangwu@163.com', '18', '1', '2018-07-09 14:37:34'), 
('8', 'zhaoliu', 'zhaoliu@163.com', '22', '1', '2018-07-11 18:29:24'), 
('9', 'fengqi', 'fengqi@163.com', '19', '1', '2018-07-11 18:29:32');


DROP TABLE IF EXISTS `tbl_userinfo`;
CREATE TABLE `tbl_userinfo` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `address` varchar(255) DEFAULT NULL,
  `user_id` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_userId` (`user_id`)
) ENGINE=InnoDB AUTO_INCREMENT=7 DEFAULT CHARSET=utf8;

INSERT INTO `tbl_userinfo` VALUES 
('1', '上海市', '1'), 
('2', '北京市', '2'), 
('3', '杭州', '3'), 
('4', '深圳', '4'), 
('5', '广州', '5'), 
('6', '海南', '6');
```

二：五百万数据插入

上面插入几条测试数据，在使用索引时还需要插入更多的数据作为测试数据，下面就通过存储过程插入500W条数据作为测试数据
```
-- 修改mysql默认的结束符号，默认是分号；但是在函数和存储过程中会使用到分号导致解析不正确
delimiter $$

-- 随机生成一个指定长度的字符串
create function rand_string(n int) returns varchar(255) 
begin 
 # 定义三个变量
 declare chars_str varchar(100) default 'abcdefghijklmnopqrstuvwxyzABCDEFJHIJKLMNOPQRSTUVWXYZ';
 declare return_str varchar(255) default '';
 declare i int default 0;

 while i < n do 
   set return_str = concat(return_str, substring(chars_str, floor(1+rand()*52), 1));
   set i = i + 1;
 end while;
 return return_str;
end $$

-- 创建插入的存储过程
create procedure insert_user(in start int(10), in max_num int(10))
begin
    declare i int default 0; 
    set autocommit = 0;  
    repeat
        set i = i + 1;
        insert into tbl_user values ((start+i) ,rand_string(8), concat(rand_string(6), '@random.com'), 1+FLOOR(RAND()*100), 3, now());
        until i = max_num
    end repeat;
   commit;
end $$

-- 将命令结束符修改回来
delimiter ;

-- 调用存储过程，插入500万数据，需要等待一会时间，等待执行完成
call insert_user(100001,5000000);
-- Query OK, 0 rows affected (7 min 49.89 sec) 我的Macbook Pro i5 8G内存用了8分钟才执行完

select count(*) from tbl_user;
```
三：使用索引和不使用索引的比较

没有添加索引前一个简单的查询用了1.79秒 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716151954411)

创建索引，然后再查询可以看到耗时0.00秒，这就是索引的威力 
![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716153459590)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716161521617)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716160619186)

# 四：explain命令

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716153546215)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716162942275)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071617361444-1572452000359)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716163329767)



explain命令用于查看sql执行时是否使用了索引，是优化SQL语句的一个非常常用而且非常重要的一个命令, 上面中的key字段表示查询使用到的索引即使用了idx_username索引

id: SELECT识别符。这是SELECT的查询序列号
select_type: 查询类型 
simple: 简单表即不适用表连接或者子查询
primary: 主查询，即外层的查询
subquery: 子查询内层第一个SELECT，结果不依赖于外部查询
dependent subquery: 子查询内层第一个
select: 依赖于外部查询
union: UNION语句中第二个SELECT开始后面所有SELECT
union result union 中合并结果
DERIVED
table：查询的表
partitions
type：扫描的方式，all表示全表扫描 
all : 全表扫描
index: 扫描所有索引
range: 索引范围扫描,常见于< <=、>、>=、between、
const: 表最多有一个匹配行, 常见于根据主键或唯一索引进行查询
system: 表仅有一行(=系统表)。这是const联接类型的一个特例
ref
possible_keys: 该查询可以利用的索引，可能同一个查询有多个索引可以使用，如果没有任何索引显示null
key: 实际使用到的索引，从Possible_key中所选择使用索引，当有多个索引时，mysql会挑出一个最优的索引来使用
key_len: 被选中使用索引的索引长度
ref: 
多表连接时的外键字段
const
rows: 估算出结果集行数，该sql语句扫描了多少行，可能得到的结果,MySQL认为它执行查询时必须检查的行数
filtered:
Extra： 额外重要的信息 
no tables: Query语句中使用FROM DUAL 或不含任何FROM子句
using filesort : 使用文件排序，最好能避免这种情况
Using temporary: 某些操作必须使用临时表，常见 GROUP BY ; ORDER BY
Using where: 不用读取表中所有信息，仅通过索引就可以获取所需数据;
Using join buffer (Block Nested Loop)
Using index condition
Using sort_union(索引名)
查看索引的使用情况： 
show status like ‘Handler_read%’;

Handler_read_key: 越高越好 
Handler_read_rnd_next:越低越好
![这里写图片描述](https://img-blog.csdn.net/20180717135547789)

查询优化器：

重新定义表的关联顺序(优化器会根据统计信息来决定表的关联顺序)
将外连接转化成内连接(当外连接等于内连接)
使用等价变换规则（如去掉1=1）
优化count()、min()、max()
子查询优化
提前终止查询
in条件优化
mysql可以通过 EXPLAIN EXTENDED 和 SHOW WARNINGS 来查看mysql优化器改写后的sql语句 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180717141241102)

# 五：走索引的情况和不走索引的情况

#### 1. in走索引

in操作能避免则避免，若实在避免不了，需要仔细评估in后边的集合元素数量，控制在1000个之内。 


![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071615493442)

#### 2. 范围查询走索引

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716160826200)

#### 3. 模糊查询只有左前缀使用索引

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071616115967)

#### 4. 反向条件不走索引 != 、 <> 、 NOT IN、IS NOT NULL

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071615473391)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719163958221.png)
```
# 常见的对not in的优化，使用左连接加上is null的条件过滤
SELECT id, username, age FROM tbl_user WHERE id NOT IN (SELECT user_id FROM tbl_order);


SELECT u.id, u.username, u.age
FROM tbl_user u
LEFT JOIN tbl_order o ON u.id = o.user_id
WHERE o.user_id IS NULL;
————————————————
版权声明：本文为CSDN博主「vbirdbest」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/vbirdbest/article/details/81065566
```
#### 5. 对条件计算(使用函数或者算数表达式)不走索引

使用函数计算不走索引，无论是对字段使用了函数还是值使用了函数都不走索引，解决办法通过应用程序计算好，将计算的结果传递给sql，而不是让数据库去计算 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716155649934)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716160715540)

id是主键，id/10使用了算数表达式不走索引 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719170715106)

#### 6. 查询时必须使用正确的数据类型

如果索引字段是字符串类型，那么查询条件的值必须使用引号，否则不走索引 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719172455944)

#### 7. or 使用索引和不使用索引的情况

or 只有两边都有索引才走索引，如果都没有或者只有一个是不走索引的 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071616252136)

#### 8. 用union少用or

尽量避免使用or，因为大部分or连接的两个条件同时都进行索引的情况几率比较小，应使用uninon代替，这样能走索引的走索引，不能走索引的就全表扫描。 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180717094707293)

#### 9. 能用union all就不用union

union all 不去重复，union去重复，union使用了临时表，应尽量避免使用临时表

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071709495310) 

#### 10. 复合索引

对于复合索引，如果单独使用右边的索引字段作为条件时不走索引的。即复合索引如果不满足最左原则leftmost不会走复合索引 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716165420937)

11. 覆盖索引

覆盖索引是select的数据列只用从索引中就能够取得，不必读取数据行，换句话说查询列要被所建的索引覆盖。 
当能通过读取索引就可以得到想要的数据，那就不需要读取行了。一个索引包含了（或覆盖了）满足查询结果的数据就叫做覆盖索引。

它包括在查询里的Select、Join和Where子句用到的所有列（即建索引的字段正好是覆盖查询条件中所涉及的字段，也即，索引包含了查询正在查找的数据）

覆盖索引： 根据关键字就能够直接获取查询所需要的所有数据，不必要读取数据行的数据，所有数据是指where、select从句、order by、 group by从句的值

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180717110236469)

如果索引字段是字符串那么查询条件必须加引号，但是如果查询的列都在索引中，即使不满足走索引的条件，此时也会使用索引。示例中order_code=666666,是数字类型，没有加引号，按说是不走索引的，但是select * 而test表只有两个字段，id和order_code而这两个字段都创建了索引，这种情况也会走索引 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719172845878)

12. order by

mysql有两种排序方式：

通过有序索引顺序扫描直接返回有序数据，通过explain分析显示Using Index,不需要额外的排序，操作效率比较高。
通过对返回数据进行排序，也就是Filesort排序，所有不是通过索引直接返回排序结果的都叫Filesort排序。Filesort是通过相应的排序算法将取得的数据在sort_buffer_size系统变量设置的内存排序中进行排序，如果内存装载不下，就会将磁盘上的数据进行分块，再对各个数据块进行排序，然后将各个块合并成有序的结果集
order by 使用索引的严格要求：

索引的顺序和order by子句的顺序完全一致
索引中所有列的方向(升续、降续)和order by 子句完全一致
当多表连接查询时order by中的字段必须在关联表中的第一张表中
如果有 order by 的场景，请注意利用索引的有序性。order by 最后的字段是组合 索引的一部分，并且放在索引组合顺序的最后，避免出现 file_sort 的情况，影响查询性能。 
正例:where a=? and b=? order by c; 索引:a_b_c 
反例:索引中有范围查找，那么索引有序性无法利用，如:WHERE a>10 ORDER BY b; 索引 a_b 无法排序
![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716171436609)

order by如果根据多个值进行排序，那么排序方式必须保持一致，要么同时升续，要么同时降续，排序方式不一致不走索引 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180717111450206)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180722130037761)

13. group by

默认情况下，group by column； 有两个作用，第一个就是根据指定的列进行分组，第二作用group by 不但分组，而且还为分组中的数据按照列来排序，如果分组的字段创建了索引，那么排序也没什么毕竟排序走索引也很快 
但是如果group by指定的列没有走索引，而我们通常情况下只对分组中的数据进行统计，例如对分组中的数据求和，通常顺序无关紧要，此时就要关闭group by 的排序功能，使用Order By NULL;来关闭排序功能，避免排序对性能的影响。 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716172547895)

14. 分页limit

分页查询一般会全表扫描，优化的目的应尽可能减少扫描；

第一种思路：在索引上完成排序分页的操作，最后根据主键关联回原表查询原来所需要的其他列。这种思路是使用覆盖索引尽快定位出需要的记录的id，覆盖索引效率高些

第二中思路：limit m,n 转换为 n

之前分页查询是传pageNo页码, pageSize分页数量， 
当前页的最后一行对应的id即last_row_id，以及pageSize，这样先根据条件过滤掉last_row_id之前的数据，然后再去n挑记录,此种方式只能用于排序字段不重复唯一的列，如果用于重复的列，那么分页数据将不准确

当只一行数据使用limit 1 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716183200779)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071621045124)

多表连接查询连接条件(也就是外键必须创建索引，否则大数据查询直接卡死) 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716180515933)

如果全表扫描比使用索引快，就不会使用索引，比如 表的数量很少或者满足条件的数据量比较大也不走索引, 查询数据库记录时，查询到的条目数尽量小，当通过索引获取到的数据库记录> 数据库总记录的1/3时，SQL将有可能直接全表扫描，索引就失去了应有的作用。 

![这里写图片描述](https://img-blog.csdn.net/20180716221914226)

where条件将能过滤掉多的条件写在前面，过滤掉少部分的数据写在后面，这样先排除一大部分不满足条件的数据，然后剩下一小部分数据，然后再从中找出满足条件的记录 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180717100714929)

数据类型不匹配是不会走索引，例如对字符串类型创建索引，where条件值却没有使用引号，就不走索引,join语句中join条件字段类型不一致的时候MYSQL无法使用索引

15 in和exists

查询所有下过订单的用户(tbl_user 500w条数据，tbl_order 3条数据), 有两种方式，一种使用in另一种使用exists，但是两者效率相差很大 ![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180720203116349)

in的伪代码：
```
/**
 * tbl_user 有500w条数据
 * tbl_order 有3条数据
 * 循环users.size次并在内存中比较包含
 * in: 总共执行2次SQL
 * @return
 */
public List<User> in() {
    // SELECT * FROM tbl_user WHERE id IN (SELECT user_id FROM tbl_order)
    // 首先执行内层子查询，再执行外层查询, 总共执行两个查询
    List<Long> userIds = query("SELECT user_id FROM tbl_order");
    List<User> users = query("SELECT * FROM tbl_user");

    List<User> result = new ArrayList<>();
    for(int i = 0; i < users.size(); i++) {
        User user = users.get(i);
        // 在内存中比较是否包含
        for(int j = 0; j < userIds.size(); j++) {
            if (userIds[j] == user.getId()) {
                result.add(user);
            }
        }
    }

    return result;
}
```

```
SELECT * FROM A WHERE id IN (SELECT a_id FROM B) 当A表中的数据量远大于B表中的数据量时使用in
```

查询用户id大于10的用户的订单信息 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180720210015547)

```
/**
 * tbl_user 3条数据
 * tbl_order 为500W条数据
 *
 * 使用exists会查询1一次tbl_user表并循环users.size次(3次)查询tbl_order表，总共就4次查询
 *
 * @return
 */
public List<User> exists() {
    // SELECT * FROM tbl_user u WHERE exists(SELECT 1 FROM tbl_order o WHERE o.user_id = u.id)
    List<User> users = query("SELECT * FROM tbl_user");
    List<User> result = new ArrayList<>();
    for(int i = 0; i < users.size(); i++) {
        User user = users.get(i);
        // 每循环一次就会操作一次数据库
        boolean isExists = query("SELECT 1 FROM tbl_order o WHERE o.user_id =", user.getId());
        if (isExists) {
            result.add(user);
        }
    }

    return result;
}
```

从以上逻辑可以看到exists每次循环一下外表都要查询一下内表，即使外表数量很少，但是每循环一次外表都要去查询一个大表，这样的效率是不高的，除非子查询走索引，最好走覆盖索引。

SELECT * FROM B o WHERE exists(SELECT 1 FROM A WHERE o.user_id > 10) 网上说当B表的数据量小A表的数据量很大时用exists。上面那个例子没有测试出来，实际优化时in和exists具体那个好根据具体执行情况来选择，也不能一味的就说某种情况下用这个就比那个快
16. 强制索引

当查询时不走索引时可以通过force index 强制mysql使用指定的索引，一般情况下如果mysql不走索引它是认为全表扫描会更快些，可以通过强制走索引看一下查询时间，如果强制索引效果更好，查询速度更快就使用强制索引，如果强制索引没有明显效果就没必要使用了。

mysql强制使用索引:force index(索引名或者主键PRI)
```
-- 强制使用主键索引
select * from table force index(PRI);

-- 强制使用索引"idx_xxx"
select * from table force index(idx_xxx);

-- 强制使用索引"PRI"和"idx_xxx"
select * from table force index(PRI,idx_xxx);
```
mysql禁止某个索引：ignore index(索引名或者主键PRI)
```
-- 禁止使用主键
select * from table ignore index(PRI)

-- 禁止使用索引"idx_xxx"
select * from table ignore index(idx_xxx);

-- 禁止使用索引"PRI","idx_xxx"
select * from table ignore index(PRI,idx_xxx);
```
六：其它优化

1.禁止使用select *，需要什么字段就去取哪些字段 
![这里写图片描述](https://img-blog.csdn.net/20180717112155412)

2..超过三个表禁止join。需要join的字段数据类型必须绝对一致;多表关联查询时，保证被关联的字段需要有索引。说明:即使双表 join 也要注意表索引、SQL 性能。尽可能避免复杂的join和子查询。尽量使用左右连接，少使用内连接。永远小结果集驱动大结果集(这点mysql会自动优化)

3.不要使用count(列名)或 count(常量)来替代 count()，count()是SQL92定义的标准统计行数的语法，跟数据库无关，跟 NULL和非NULL无关。 说明:count(*)会统计值为NULL 的行，而count(列名)不会统计此列为NULL值的行。

4.不得使用外键与级联，一切外键概念必须在应用层解决。

5.禁止使用存储过程，存储过程难以调试和扩展，更没有移植性。避免使用存储过程、触发器。

6.使用表连接来优化子查询。尽量避免使用子查询，建议将子查询转换成关联查询，子查询的效率并没有连接join查询快(并不绝对)，连接查询之所以更有效率一些，是因为mysql不需要再内存中创建临时表来完成这个逻辑上需要两个步骤的查询工作。对于多表连接，如果连接条件创建索引效率更高。

7.对于列类型是字符串，值必须使用单引号括住，如果没有引号引住就不走索引， 结论：字符串必须使用‘’引住

8.拒绝大SQL，拆分成小SQL

9.优先优化高并发的SQL，而不是执行频率低某些“大”SQL
10.在所有的存储过程和触发器的开始处设置 SET NOCOUNT ON ，在结束时设置 SET NOCOUNT OFF 。无需在执行存储过程和触发器的每个语句后向客户端发送 DONE_IN_PROC 消息。
11.减少 IO 次数 
IO永远是数据库最容易瓶颈的地方，这是由数据库的职责所决定的，大部分数据库操作中超过90%的时间都是 IO 操作所占用的，减少 IO 次数是 SQL 优化中需要第一优先考虑，当然，也是收效最明显的优化手段。
12.降低 CPU 计算 
除了 IO 瓶颈之外，SQL优化中需要考虑的就是 CPU 运算量的优化了。order by, group by,distinct … 都是消耗 CPU 的大户（这些操作基本上都是 CPU 处理内存中的数据比较运算）。当我们的 IO 优化做到一定阶段之后，降低 CPU 计算也就成为了我们 SQL 优化的重要目标

13.大表的数据修改最好要分批处理，例如1000万行的记录删除更新 100万行记录，可以一次只删除更新5000行记录，暂停几秒，然后再执行剩下的数据。 
如何修改大表的表结构：

- 1.先在从服务器上修改，然后将从服务器改为主服务器，然后再从主服务器上修改，然后再切换回来，此种方式需要切换主从，有一定的风险。
- 2.在主服务器上创建一张新表，将老表的数据迁移到新表，创建一个触发器，将老表的新数据同步到新表中，对老表加一个排它锁，重新命名新表和老表，然后删除掉老表（整个操作过程比较复杂，可以通过工具来实现）

```
pt-online-schema-change 
     --alter="modify c varchar(150) not null default '' " 
     --user=root 
     --password=123456 
     D=数据库名,t=表名 
     --charset=utf8 
     --execute 
```

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180722170851408)

当进行多表连接查询时， [驱动表] 的定义为： 
1）指定了联接条件时，满足查询条件的记录行数少的表为[驱动表] 
2）未指定联接条件时，行数少的表为[驱动表]（Important!）

MySQL表关联的算法是 Nest Loop Join，是通过驱动表的结果集作为循环基础数据，然后一条一条地通过该结果集中的数据作为过滤条件到下一个表中查询数据，然后合并结果。

七：SQL 提示(sql hint)

在sql语句中加入一些人为的提示达到绣花操作的目的。

```
-- SQL_BUFFER_RESULT: 强制生成一个临时结果集，只要临时结果集生成后，所有表上的锁定均被释放。这能再遇到表锁定问题时或者
-- 要花很长时间将结果传给客户端时有所帮助，因为可以尽快释放锁资源。
SELECT SQL_BUFFER_RESULT * FROM tbl_user;

-- mysql参考的索引列表，不在这个配置内mysql就不去使用
SELECT * FROM tbl_user USE INDEX (idx_username);

-- 让mysql或略掉指定的索引列表
SELECT * FROM tbl_user IGNORE INDEX (idx_username);

-- 有些情况下mysql会找出最快的查询方式，如果索引没有明显的提速，可能会进行全表扫描，
-- 可以强制mysql使用指定的索引，但是之于效率可能没有全表扫描的好
SELECT * FROM tbl_user FORCE INDEX (idx_username);
```

正则表达条件 SELECT * FROM tbl_user WHERE email REGEXP ‘@126.com$’;

------

# 八：show profile

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180722162146316)

show profile 可以更清楚的了SQL执行的过程

show profile 和 show profiles 语句可以展示当前会话(退出session后,profiling重置为0) 中执行语句的资源使用情况. 默认情况下profiling是关闭的.

show profiles :列表,显示最近发送到服务器上执行的语句的资源使用情况.显示的记录数由变量:profiling_history_size 控制,默认15条. 
show profile: 展示最近一条语句执行的详细资源占用信息,默认显示 Status和Duration两列. 
show profile 还可根据 show profiles 列表中的 Query_ID ,选择显示某条记录的性能分析信息.

```
-- 查看profiling是否开启
SHOW PROFILES;
SHOW VARIABLES LIKE '%profiling%';
SELECT @@profiling;
SELECT @@have_profiling;

-- 开启profiling
SET profiling=1;

-- 执行SQL
SELECT * FROM tbl_user WHERE id > 1000000 ORDER BY id ASC LIMIT 2;

SHOW PROFILES;

SHOW PROFILE for query 1;

SHOW PROFILE cpu, swaps for query 1;
```

语法结构:

SHOW PROFILE [type [, type] ... ] [FOR QUERY n] [LIMIT row_count [OFFSET offset]]

type:

- ALL

- BLOCK IO

- CONTEXT SWITCHES

- CPU

- IPC

- MEMORY

- PAGE FAULTS

- SOURCE

- SWAPS

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716212539205)

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180716212553905)

九：找出项目中执行比较慢的sql语句

MySQL慢查询日志记录下所有执行超过long_query_time时间的SQL语句，帮你找到执行慢的SQL(包括查询语句、修改语句、已经回滚的sql)，方便我们对这些SQL进行优化。 默认情况下，mysql没有启用慢查询日志，如果需要启动需要在my.cnf配置文件中配置开启

show [session|global] status like 'xxx' 
默认是session会话，即取出当前窗口的执行，global值从mysql启动到现在
```
-- 查看所有状态
show status;

-- 数据库启动运行的时间
show status like 'uptime'; 
-- 连接次数
show status like 'connections';  
-- 执行的查询次数(通过这几个参数就很容易的知道数据库应用是插入更新为主，还是查询为主，以及各种类型的sql执行比例)
show status like 'com_select'; 
show status like 'com_insert';
show status like 'com_update';
show status like 'com_delete';


-- 查询事务提交回滚情况   
show status like 'com_commit';  
show status like 'com_rollback';    

-- 慢查询相关变量
show variables like '%slow%';

-- 慢查询次数（不仅记录查询，而是记录执行时间超过默认时间的操作，增删改都有可能属于慢查询）
show status like 'slow_queries'; 

-- mysql默认10秒才算慢查询
show variables like 'long_query_time';
-- 修改成1秒
set long_query_time = 1；
```

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719144110590)

/etc/my.cnf
```
# Exmple MySQL config file for medium systems.  
#  
# This is for a system with little memory (32M - 64M) where MySQL plays  
# an important part, or systems up to 128M where MySQL is used together with  
# other programs (such as a web server)  
#  
# MySQL programs look for option files in a set of  
# locations which depend on the deployment platform.  
# You can copy this option file to one of those  
# locations. For information about these locations, see:  
# http://dev.mysql.com/doc/mysql/en/option-files.html  
#  
# In this file, you can use all long options that a program supports.  
# If you want to know which options a program supports, run the program  
# with the "--help" option.  
# The following options will be passed to all MySQL clients  
[client]
default-character-set=utf8
#password   = your_password  
port        = 3306  
socket      = /tmp/mysql.sock   
# Here follows entries for some specific programs  
# The MySQL server  
[mysqld]
character-set-server=utf8
init_connect='SET NAMES utf8
port        = 3306  
socket      = /tmp/mysql.sock  
skip-external-locking  
key_buffer_size = 16M  
max_allowed_packet = 1M  
table_open_cache = 64  
sort_buffer_size = 512K  
net_buffer_length = 8K  
read_buffer_size = 256K  
read_rnd_buffer_size = 512K  
myisam_sort_buffer_size = 8M  
character-set-server=utf8  
init_connect='SET NAMES utf8' 
# Don't listen on a TCP/IP port at all. This can be a security enhancement,  
# if all processes that need to connect to mysqld run on the same host.  
# All interaction with mysqld must be made via Unix sockets or named pipes.  
# Note that using this option without enabling named pipes on Windows  
# (via the "enable-named-pipe" option) will render mysqld useless!  
#   
#skip-networking  

# Replication Master Server (default)  
# binary logging is required for replication  
log-bin=mysql-bin  

# binary logging format - mixed recommended  
binlog_format=mixed  

# required unique id between 1 and 2^32 - 1  
# defaults to 1 if master-host is not set  
# but will not function as a master if omitted  
server-id   = 1  

# Replication Slave (comment out master section to use this)  
#  
# To configure this host as a replication slave, you can choose between  
# two methods :  
#  
# 1) Use the CHANGE MASTER TO command (fully described in our manual) -  
#    the syntax is:  
#  
#    CHANGE MASTER TO MASTER_HOST=<host>, MASTER_PORT=<port>,  
#    MASTER_USER=<user>, MASTER_PASSWORD=<password> ;  
#  
#    where you replace <host>, <user>, <password> by quoted strings and  
#    <port> by the master's port number (3306 by default).  
#  
#    Example:  
#  
#    CHANGE MASTER TO MASTER_HOST='125.564.12.1', MASTER_PORT=3306,  
#    MASTER_USER='joe', MASTER_PASSWORD='secret';  
#  
# OR  
#  
# 2) Set the variables below. However, in case you choose this method, then  
#    start replication for the first time (even unsuccessfully, for example  
#    if you mistyped the password in master-password and the slave fails to  
#    connect), the slave will create a master.info file, and any later  
#    change in this file to the variables' values below will be ignored and  
#    overridden by the content of the master.info file, unless you shutdown  
#    the slave server, delete master.info and restart the slaver server.  
#    For that reason, you may want to leave the lines below untouched  
#    (commented) and instead use CHANGE MASTER TO (see above)  
#  
# required unique id between 2 and 2^32 - 1  
# (and different from the master)  
# defaults to 2 if master-host is set  
# but will not function as a slave if omitted  
#server-id       = 2  
#  
# The replication master for this slave - required  
#master-host     =   <hostname>  
#  
# The username the slave will use for authentication when connecting  
# to the master - required  
#master-user     =   <username>  
#  
# The password the slave will authenticate with when connecting to  
# the master - required  
#master-password =   <password>  
#  
# The port the master is listening on.  
# optional - defaults to 3306  
#master-port     =  <port>  
#  
# binary logging - not required for slaves, but recommended  
#log-bin=mysql-bin  

# Uncomment the following if you are using InnoDB tables  
#innodb_data_home_dir = /usr/local/mysql/data  
#innodb_data_file_path = ibdata1:10M:autoextend  
#innodb_log_group_home_dir = /usr/local/mysql/data  
# You can set .._buffer_pool_size up to 50 - 80 %  
# of RAM but beware of setting memory usage too high  
#innodb_buffer_pool_size = 16M  
#innodb_additional_mem_pool_size = 2M  
# Set .._log_file_size to 25 % of buffer pool size  
#innodb_log_file_size = 5M  
#innodb_log_buffer_size = 8M  
#innodb_flush_log_at_trx_commit = 1  
#innodb_lock_wait_timeout = 50
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data  
slow_query_log=true
slow_query_log_file = /usr/local/mysql/data/slow.log
long_query_time=1

[mysqldump]  
quick  
max_allowed_packet = 16M  

[mysql]  
no-auto-rehash  
# Remove the next comment character if you are not familiar with SQL  
#safe-updates  
default-character-set=utf8   

[myisamchk]  
key_buffer_size = 20M  
sort_buffer_size = 20M  
read_buffer = 2M  
write_buffer = 2M  

[mysqlhotcopy]  
interactive-timeout
```

windows是my.ini文件，mac是/etc/my.cnf，在高版本中没有，需要手动创建，找出慢sql需要在my.cnf配置如下参数

basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
slow_query_log=true # 开启慢查询
slow_query_log_file = /usr/local/mysql/data/slow.log # 指定慢查询日志的存储路径和文件
long_query_time=1 # 指定记录慢查询的时间阀值
配置完成后重启mysql服务器

```
# 关闭mysql
sudo /usr/local/mysql/support-files/mysql.server stop

# 启动mysql
sudo /usr/local/mysql/support-files/mysql.server start
# 如果启动报错使用下面命令来解决
ERROR! The server quit without updating PID file (/usr/local/mysql/data/MacOSX.local.pid).
sudo chown -R mysql:mysql /usr/local/var/mysql
sudo /usr/local/mysql/support-files/mysql.server start
————————————————
版权声明：本文为CSDN博主「vbirdbest」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/vbirdbest/article/details/81065566
```

执行一个耗时的sql，然后去/usr/local/mysql/data/slow.log目录查看日志中已经记录了该sql 
SELECT * FROM tbl_user WHERE email = ‘mengday@163.com’;

十：记录不走索引的SQL

mysql不但能记录慢查询对应的sql，也能记录没有使用索引的sql

```
-- 查看是否开启记录没有使用索引的功能
show variables like '%not_using_index%';

-- 开启记录没有使用索引的记录
set @@global.log_queries_not_using_indexes=on;

-- 执行一个不走索引的查询
EXPLAIN SELECT id, username FROM tbl_user WHERE username = 'admin' OR email = 'admin@163.com';

SELECT id, username FROM tbl_user WHERE username = 'admin' OR email = 'admin@163.com';
```

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/2018071915070227)

不走索引的日志仍然记录在慢查询对应的日志文件，这里配置的是/usr/local/mysql/data/slow.log 

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719150717105)

# 十一： 分析慢查询SQL

#### 1. 可以通过mysqldumpslow命令来分析慢查询的SQL

```
mysqldumpslow -s r -t 10 /usr/local/mysql/data/slow.log
```

-s 排序：自定按那种排序方式输出结果

- c: 总次数
- t : 总时间
- l：锁的时间
- r: 总数据行
- at,al,ar: t，l, r 平均数， 例如at =总时间/总次数

-t top: 指定取前几条作为输出结果

![这里写图片描述](image/良心文章-SQL优化(好文章希望更多人能学到)/20180719151055869)

#### 2. 使用第三方工具pt-query-digest

```
--explain:会生成每个SQL对应的执行计划
pt-query-digest --explain h=127.0.0.1,u=root,p=123456  /usr/local/mysql/data/slow.log;
```

#### 3. 实时获取有性能的sql

```
-- 实时获取有性能的sql
SELECT  id,  user, host, db, command, time, state, info FROM information_schema.`PROCESSLIST`;
```



————————————————
版权声明：本文为CSDN博主「vbirdbest」的原创文章，遵循 CC 4.0 BY-SA 版权协议，转载请附上原文出处链接及本声明。
原文链接：https://blog.csdn.net/vbirdbest/article/details/81065566