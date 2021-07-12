# :star:mysql

mysql学习笔记。。。:book::book::book:

## 索引

索引是帮助mysql高效获取数据的排好序的数据结构

### 数据结构

![image-20210702160121818](README.assets\image-20210702160121818.png)

#### 二叉树

左边的子节点小于父节点，右边子节点大于父节点

缺点：当数据出现单边增长时，会退化成链表结构，达不到优化效果

![image-20210702161009634](README.assets/image-20210702161009634.png)

#### 红黑树

平衡二叉树，会随着数据的插入自动做平衡处理，平衡树的左右高度

![image-20210702162700935](README.assets/image-20210702162700935.png)

缺点：数据量大时，树的高度依然不可控

#### Hash（索引方法）

- 对索引的key进行一次hash计算就可以定位出数据存储的位置
- 很多时候Hash索引要比B+Tree索引更高效
- 缺点：仅能满足“=”，“IN”，不支持范围查询；hash冲突问题

![image-20210707213514489](README.assets/image-20210707213514489.png)

#### B-Tree

- 叶子节点具有相同的深度，叶子节点的指针为空
- 所有索引元素不重复
- 节点中的数据索引从左到右递增排列

![image-20210702172350609](README.assets/image-20210702172350609.png)

#### B+Tree（索引方法）

B-Tree变种

- 非叶子节点不存储data，只存储索引（冗余），可以放更多的索引
- 叶子节点包含所有的索引字段
- 叶子节点用指针连接，提高区间访问的性能

![image-20210702172715980](README.assets/image-20210702172715980.png)

tips:

​	mysql默认页大小16kb，假如索引字段使用bigint类型(8byte)，下一个子节点的地址(6byte)，一个非叶子节点约可以存储1170个索引*16384/(8+6)=1170*

可以修改默认页大小，但是不建议。

![image-20210702173900809](README.assets/image-20210702173900809.png)



### 存储引擎

![image-20210707194401860](README.assets/image-20210707194401860.png)

#### MyISAM

- 非聚集索引-叶子节点存放索引和完整数据行的磁盘地址

- MyISAM索引文件和数据文件是分离的

![image-20210707195247159](README.assets/image-20210707195247159.png)

![image-20210707200212975](README.assets/image-20210707200212975.png)

#### InnoDB

- 聚集索引-叶子节点包含了完整的数据行记录
  - 聚集索引的查询速度略优于非聚集索引
- 表数据文件本身就是按照B+Tree组织的一个索引结构文件
- 建议InnoDB表必须建立主键，并且推荐使用整形的自增主键
  - 主键唯一，为了组织唯一索引B+Tree，必须要有数据唯一列。如果没有主键列，mysql会从表中找一个数据唯一列来组织B+Tree，如果没找到，mysql会建立一个隐藏的数据唯一列（rowid）
  - 整形占用空间小，且利于比较大小
  - B+Tree元素本身是从左到右顺序排列，自增主键可以避免B+Tree重新排列，分裂
- 非主键索引（二级索引，非聚集索引）结构的叶子节点存储的是主键值
  - 节约空间，数据一致性
  - 查询时先定位到主键，再通过主键索引去拿到整行数据（回表）

![image-20210707200625588](README.assets/image-20210707200625588.png)

![image-20210707203955777](README.assets/image-20210707203955777.png)tips:一张表只有一个聚集索引，就是用来组织这棵B+树的索引，通常就是主键

### 联合索引

![image-20210707220555215](README.assets/image-20210707220555215.png)

#### 索引最左前缀原理

查询条件必须符合索引字段从左到右的顺序，不能跳过前面的字段

tips:mysql查询器会优化sql语句的查询字段顺序，即使字段顺序不对也会走索引，前提是不能跳过

![image-20210707220823201](README.assets/image-20210707220823201.png)

为什么？

跳过前面字段的话，无法确定元素是顺序排列的，违背了索引基本要求

![image-20210707221722947](README.assets/image-20210707221722947.png)

### Explain

![image-20210708215659829](README.assets/image-20210708215659829.png)

#### Explain中的列

- id

  - id列的编号是select的序列号，有几个select就有几个id，并且id的顺序是按照select出现的顺序增长的

  - id列越大执行优先级越高

- select_type

  - simple:简单查询。不包含子查询和union
  - primary:复杂查询中最外层的select
  - subquery:包含在select中的子查询（不在from子句中）
  - derived:包含在from子句中的子查询。mysql会将结果存放在一个临时表中，也成为衍生表（派生 derived）
  - union:在union中的第二个和随后的select

- table

  表示explain的一行正在访问的表

  - 当from子句有子查询时，table列是`<derivedN>`格式，表示当前查询依赖id=N的查询，于是先执行id=N的查询
  - 当有union时，UNION RESULT的table列的值为`<union1,2>`，1和2表示参与union的select行id

- type

  表示关联类型或访问类型，即mysql决定如何查找表中的行，查找数据行记录的大概范围。

  依次从最优到最差分别为：**system>const>eq_ref>ref>range>index>ALL**

  一般来说，得保证查询达到range级别，最好达到ref

  - NULL

    mysql能够在优化阶段分解查询语句，在执行阶段用不着再访问表或索引。例如：在索引列中选取最小值，可以单独查找索引来完成，不需要在执行时访问表

    ![image-20210712133933247](README.assets/image-20210712133933247.png)

  - const,system

    mysql能对查询的某部分进行优化并将其转化成一个常量（可以看show warnings）。用于primary key或unique key的所有列与常数比较时，所以表最多有一个匹配行，读取1次，速度比较快。**system是const的特例**，表里只有1条元素匹配时为system

    ![image-20210712134547676](README.assets/image-20210712134547676.png)

    tips:

    ​	dual:mysql中的空表

  - eq_ref

    primary key或unique key索引的所有部分被连接使用，最多只会返回一条符合条件的记录。这可能是在const之外最好的连接类型，简单的select查询不会出现这种type

    ![image-20210712141129466](README.assets/image-20210712141129466.png)

  - ref

    相比eq_ref，不使用唯一索引，而是使用普通索引或者唯一索引的部分前缀，索引要和某个值相比较，可能会找到多个符合条件的行

    ![image-20210712142029409](README.assets/image-20210712142029409.png)

  - range

    范围扫描通常出现在in(),between,>,<,>=,<=等操作中，使用一个索引来检索给定范围的行

    ![image-20210712142656380](README.assets/image-20210712142656380.png)

  - index

    全索引扫描，一般是扫描某个二级索引，这种扫描不会从索引树根节点开始快速查找，而是直接对二级索引的叶子节点遍历和扫描，速度还是比较慢的，这种查询一般为使用覆盖索引，二级索引一般比较小，所以通常比ALL快一些

    ![image-20210712144142485](README.assets/image-20210712144142485.png)

  - ALL

    即全表扫描，扫描聚簇索引的所有叶子节点。通常情况下这需要增加索引来进行优化

    ![image-20210712144623780](README.assets/image-20210712144623780.png)

- possible_keys

  显示查询可能使用哪些索引来查找

  explain时可能出现 possible_keys有值，而key显示NULL的情况，这种情况是因为表中数据不多，mysq认为索引对此查询帮助不大，选择了全表查询
  如果该列是NULL，则没有相关的索引。在这种情况下，可以通过检查 where子句看是否可以创造一个适当的索引来提高查询性能，然后用explain查看效果

- key

  显示mysq实际采用哪个索引来优化对该表的访问

  如果没有使用索引，则该列是NULL。如果想强制 mysql使用或忽视possible_keys列中的索引，在查询中使用 force index、 ignore index

- key_len

  这一列显示了mysq在索引里使用的字节数，通过这个值可以算出具体使用了索引中的哪些列

  举例来说，film_actor的联合索引 idx_film_acto_id由film_id和 actor_id两个int列组成，并且每个int是4字节。通过结果中的 keylen=4可推断出查询使用了第一个列: film id列来执行索引查找

  ![image-20210712162342628](README.assets/image-20210712162342628.png)

  ![image-20210712162425951](README.assets/image-20210712162425951.png)



