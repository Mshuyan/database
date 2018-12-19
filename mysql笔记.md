### 字段长度

1. mysql中，数字类型和字符串类型均可设置长度，各类型所占字节数参见[mysql数据类型](http://www.runoob.com/mysql/mysql-data-types.html)

2. 上文中，类型分为两种：

   - 所占字节数是固定的
   - 所占字节数是不固定的，需要定义数据类型时指定其长度

3. 所以指定的长度，针对上述两种情况，这个长度的含义是不同的

   - 对于本身字节数固定的类型，如`int(11)`，这个指定的长度表示`显示宽度`

     - 存储数据时所占空间与该类型所占字节数有关，而与指定的长度无关

     - 当指定字段类型时指定了该字段的`zerofull`属性，这个`显示宽度`才有用

       如果设置1个字段类型为`int(3)`并指定了`zerofill`属性，插入数据`10`，则显示为`010`，如果插入`1000`，则正常显示为`1000`

     - 综上所述，int类型指定长度时，这个长度只有在指定了该字段的`zerofill`属性并且插入的值小于这个长度时，才在显示的时候有所不同，否则没有任何用处。

   - 对于本身字节数不固定的类型，如`varchar(50)`，这个指定的长度会影响该字段的最大存储空间

### 分组后查询最新记录

+ mysql5.6

  ```sql
  select * from 
  (select * from table1 order by submit_time) as temp 
  group by user_id
  ```

+ mysql5.7

  ```sql
  select * from 
  (select * from table1 order by submit_time limit 10000000000) as temp 
  group by user_id
  ```

  > 分析如下：

  + 解决报错

    因为`mysql5.7`对`group by`进行了修改，默认的`sql_mode`如下：

    ```mysql
    mysql> select @@sql_mode;
    +-------------------------------------------------------------------------------------------------------------------------------------------+
    | @@sql_mode                                                                                                                                |
    +-------------------------------------------------------------------------------------------------------------------------------------------+
    | ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION |
    +-------------------------------------------------------------------------------------------------------------------------------------------+
    1 row in set (0.00 sec)
    ```

    而`mysql5.6`查询结果如下：

    ```mysql
    mysql> select @@sql_mode;
    +--------------------------------------------+
    | @@sql_mode                                 |
    +--------------------------------------------+
    | STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION |
    +--------------------------------------------+
    1 row in set (0.00 sec)
    ```

    可以看到，`mysql5.7`与`mysql5.6`关于`sql_mode`的区别中，包含一条：

    + 5.7比5.6多了1个`ONLY_FULL_GROUP_BY`，这个属性表示不在`group by`后的字段不能被直接查询

    这也就导致了上面`mysql5.6`的sql在`mysql5.7`中会报错

    该错误可以使用`ANY_VALUE()`消除

  + 解决查询到的记录不是想要的

    > 资料参见：[MySQL5.7排序后GROUP BY 问题](https://bbs.csdn.net/topics/391998346)第5、6楼的回复

    但是通过`ANY_VALUE()`对`mysql5.6`的sql进行修改后，发现得到的记录并不是想要的记录

    这是因为`ANY_VALUE()`取得是分组后第一条记录对应的记录的值，而子查询中的`order by`在`mysql5.7`中被忽略掉了

    解决方案是在子查询中加入`limit`，则子查询中的`order by`不会忽略，即可达到预期

+ 通用版本

  其实上面的方式都是根据`order by`和`group by`的特性投机取巧得到目的的，正常思路的语句如下：

  ```sql
  select * from table1 as t,
  (select user_id,max(submit_time) as submit_time from table1) as temp
  where t.user_id=temo.user_id and t.submit_time=temp.submit_time
  ```
