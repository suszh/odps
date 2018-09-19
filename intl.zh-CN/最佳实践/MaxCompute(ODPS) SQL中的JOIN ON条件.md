# MaxCompute\(ODPS\) SQL中的JOIN ON条件 {#concept_vms_4sv_1fb .concept}

MaxCompute\(ODPS\) SQL中，很常用的一个操作就是关联\(Join\)。

## 概述 {#section_ugb_c1w_1fb .section}

目前MaxCompute提供了以下几种Join类型：

|类型|含义|
|:-|:-|
|Inner Join|输出符合关联条件的数据|
|Left Join|输出左表的所有记录，对于右表符合关联的数据，输出右表，没有符合的，右表补null。|
|Right Join|输出右表的所有记录，对于左表符合关联的数据，输出左表，没有符合的，左表补null。|
|Full Join|输出左表和右表的所有记录，对于没有关联上的数据，未关联的另一侧补null。|
|Left Semi Join|对于左表中的一条数据，如果右表存在符合关联条件的行，则输出左表。|
|Left Anti Join|对于左表中的一条数据，如果对于右表所有的行，不存在符合关联条件的数据，则输出左表。|

**说明：** [User Defined Join](https://www.atatech.org/articles/100074)指定两个输入流，用户自己实现Join的逻辑，这里不展开讨论。

根据不同的场景，用户可以使用不同的Join类型来实现对应的关联操作。但是在实际使用过程当中，经常有用户分不清楚过滤条件在JOIN ON语句中还是在WHERE中有什么区别，或者认为他们的效果都是一样的，例如在生产的环境中经常可以看到用户写了：

```
A (LEFT/RIGHT/FULL/LEFT SEMI/LEFT ANTI) JOIN B
ON a.key = b.key and A.ds='20180101' and B.ds='20180101';
```

这里用户的本意是希望在A和B中获取某一个分区的数据进行JOIN操作，也就是：

```
(SELECT * FROM A WHERE ds='20180101') A
(LEFT/RIGHT/FULL/LEFT SEMI/LEFT ANTI)  JOIN
(SELECT * FROM B WHERE ds='20180101') B
ON a.key = b.key
```

然而针对不同的Join类型，两者可能并不等价，不仅无法将分区条件下推，导致全表扫描，而且会导致正确性问题。这里简要辨析一下过滤条件分别在：

1.  子查询的WHERE条件。
2.  JOIN ON条件。
3.  JOIN ON后的WHERE条件。

## 原理 {#section_jqg_f1w_1fb .section}

这里先说明一个JOIN和WHERE条件的计算顺序，对于：

```
(SELECT * FROM A WHERE {subquery_where_condition} A) A
JOIN
(SELECT * FROM B WHERE {subquery_where_condition} B) B
ON {on_condition}
WHERE {where_condition}
```

来说，计算顺序为：

1.  子查询中的`{subquery_where_condition}`
2.  JOIN的`{on_condition}`的条件
3.  JOIN结果集合`{where_condition}`的计算

对于不同的JOIN类型，过滤语句放在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`中，有时结果是一致的，有时候结果又是不一致的。下面分情况进行讨论。

## 实验 {#section_lvl_j1w_1fb .section}

1.  准备

    首先构造表A：

    ```
    CREATE TABLE A AS SELECT * FROM VALUES (1, 20180101),(2, 20180101),(2, 20180102) t (key, ds);
    ```

    |key|ds|
    |:--|:-|
    |1|20180101|
    |2|20180101|
    |2|20180102|

    表B：

    ```
    CREATE TABLE B AS SELECT * FROM VALUES (1, 20180101),(3, 20180101),(2, 20180102) t (key, ds);
    ```

    |key|ds|
    |:--|:-|
    |1|20180101|
    |3|20180101|
    |2|20180102|

    则他们的笛卡尔乘积为：

    |a.key|a.ds|b.key|b.ds|
    |:----|:---|:----|:---|
    |1|20180101|1|20180101|
    |1|20180101|3|20180101|
    |1|20180101|2|20180102|
    |2|20180101|1|20180101|
    |2|20180101|3|20180101|
    |2|20180101|2|20180102|
    |2|20180102|1|20180101|
    |2|20180102|3|20180101|
    |2|20180102|2|20180102|

2.  Inner Join

    结论：过滤条件在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`中都是等价的。

    Inner Join的处理逻辑是将左右表进行笛卡尔乘积，然后选择满足ON表达式的行进行输出。

    1.  第一种情况，子查询中过滤：

        ```
        SELECT A.*, B.*
        FROM
        (SELECT * FROM A WHERE ds='20180101') A
        JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key;
        ```

        非常简单，结果只有一条：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|

    2.  第二种情况，JOIN 条件中过滤：

        ```
        SELECT A.*, B.*
        FROM A JOIN B
        ON a.key = b.key and A.ds='20180101' and B.ds='20180101';
        ```

        笛卡尔积的结果有9条，满足ON条件的结果同样只有1条：

        |a.key|a.ds|b.key|b.ds|
        |1|20180101|1|20180101|

    3.  第三种情况，JOIN后的WHERE条件过滤：

        ```
        SELECT A.*, B.*
        FROM A JOIN B
        ON a.key = b.key
        WHERE A.ds='20180101' and B.ds='20180101';
        ```

        来说，笛卡尔积的结果有9条，满足ON条件`a.key = b.key`的结果有3条，分别是：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180102|2|20180102|
        |2|20180101|2|20180102|

        此时对于这个结果再进行过滤`A.ds='20180101' and B.ds='20180101'`，结果只有1条：

        |a.key|a.ds|b.key|b.ds|
        |-----|----|-----|----|
        |1|20180101|1|20180101|

        可以看到，将过滤条件放在三个不同的地方，得到了三种不同的结果。

3.  Left Join

    结论：过滤条件在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`不一定等价。

    对于左表的过滤条件，放在`{subquery_where_condition}`和`{where_condition}`是等价的。

    对于右表的过滤条件，放在`{subquery_where_condition}`和`{on_condition}`中是等价的。

    Left Join的处理逻辑是将左右表进行笛卡尔乘积，然后对于满足ON表达式的行进行输出，对于左表中不满足ON表达式的行，输出左表，右表补NULL。

    1.  第一种情况，子查询中过滤：

        ```
        SELECT A.*, B.*
        FROM
        (SELECT * FROM A WHERE ds='20180101') A
        LEFT JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key;
        ```

        过滤后，左右侧有两条，右侧有一条，结果有两条：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|NULL|NULL|

    2.  第二种情况，JOIN 条件中过滤：

        ```
        SELECT A.*, B.*
        FROM A LEFT JOIN B
        ON a.key = b.key and A.ds='20180101' and B.ds='20180101';
        ```

        笛卡尔积的结果有9条，满足ON条件的结果同样只有1条，则对于左表剩余的两条输出左表，右表补NULL：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|NULL|NULL|
        |2|20180102|NULL|NULL|

    3.  第三种情况，JOIN后的WHERE条件过滤：

        ```
        SELECT A.*, B.*
        FROM A LEFT JOIN B
        ON a.key = b.key
        WHERE A.ds='20180101' and B.ds='20180101';
        ```

        来说，笛卡尔积的结果有9条，满足ON条件`a.key = b.key`的结果有3条，分别是：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|2|20180102|
        |2|20180102|2|20180102|

        此时对于这个结果再进行过滤`A.ds='20180101' and B.ds='20180101'`，结果只有1条：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|

        可以看到，将过滤条件放在三个不同的地方，得到了三种不同的结果。

4.  Right Join

    Right Join和Left Join是类似的，只是左右表的区别。结论：过滤条件在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`不一定等价。对于右表的过滤条件，放在`{subquery_where_condition}`和`{where_condition}`是等价的。对于左表的过滤条件，放在`{subquery_where_condition}`和`{on_condition}`中是等价的。

5.  Full Join

    结论：过滤条件写在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`均不等价。

    FULL Join的处理逻辑是将左右表进行笛卡尔乘积，然后对于满足ON表达式的行进行输出，对于两侧表中不满足ON表达式的行，输出有数据的表，另一侧补NULL。

    1.  第一种情况，子查询中过滤：

        ```
        SELECT A.*, B.*
        FROM
        (SELECT * FROM A WHERE ds='20180101') A
        FULL JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key;
        ```

        过滤后，左右侧有两条，右侧有两条，结果有三条：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|NULL|NULL|
        |NULL|NULL|3|20180101|

    2.  第二种情况，JOIN 条件中过滤：

        ```
        SELECT A.*, B.*
        FROM A FULL JOIN B
        ON a.key = b.key and A.ds='20180101' and B.ds='20180101';
        ```

        笛卡尔积的结果有9条，满足ON条件的结果同样只有1条，则对于左表剩余的两条输出左表，右表补NULL。右表剩余的两条输出右表，左表补NULL：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|NULL|NULL|
        |2|20180102|NULL|NULL|
        |NULL|NULL|3|20180101|
        |NULL|NULL|2|20180102|

    3.  第三种情况，JOIN后的WHERE条件过滤：

        ```
        SELECT A.*, B.*
        FROM A FULL JOIN B
        ON a.key = b.key
        WHERE A.ds='20180101' and B.ds='20180101';
        ```

        笛卡尔积的结果有9条，满足ON条件`a.key = b.key`的结果有3条，分别是：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|2|20180102|
        |2|20180102|2|20180102|

        再对没有JOIN上的数据进行输出，另一侧补NULL，得到结果：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|
        |2|20180101|2|20180102|
        |2|20180102|2|20180102|
        |NULL|NULL|3|20180101|

        此时对于这个结果再进行过滤`A.ds='20180101' and B.ds='20180101'`，结果只有1条：

        |a.key|a.ds|b.key|b.ds|
        |:----|:---|:----|:---|
        |1|20180101|1|20180101|

        可以看到，和LEFT JOIN类似，得到了三种不同的结果。

6.  Left Semi Join

    结论：过滤条件写在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`是等价的。

    LEFT SEMI Join的处理逻辑是对于左表的每一条记录，都去和右表进行匹配，如果匹配成功，则输出左表。这里需要注意的是由于只输出左表，所以JOIN后的Where条件中不能写右侧的过滤条件。LEFT SEMI JOIN常用来实现exists的语义：

    1.  第一种情况，子查询中过滤：

        ```
        SELECT A.*
        FROM
        (SELECT * FROM A WHERE ds='20180101') A
        LEFT SEMI JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key;
        ```

        过滤后，左右侧有两条，最终符合`a.key = b.key`的只有一条：

        |a.key|a.ds|
        |:----|:---|
        |1|20180101|

    2.  第二种情况，JOIN 条件中过滤：

        ```
        SELECT A.*
        FROM A LEFT SEMI JOIN B
        ON a.key = b.key and A.ds='20180101' and B.ds='20180101';
        ```

        对于左侧的三条记录，满足ON条件的结果同样只有1条：

        |a.key|a.ds|
        |:----|:---|
        |1|20180101|

    3.  第三种情况，JOIN后的WHERE条件过滤：

        ```
        SELECT A.*
        FROM A LEFT SEMI JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key
        WHERE A.ds='20180101';
        ```

        左侧能符合ON条件的有一条：

        |a.key|a.ds|
        |:----|:---|
        |1|20180101|

        此时对于这个结果再进行过滤`A.ds='20180101'`，结果仍然保持1条：

        |a.key|a.ds|
        |-----|----|
        |1|20180101|

        可以看到，LEFT SEMI JOIN和INNER JOIN类似，无论过滤条件放在哪里，结果都是一致的。

7.  Left Anti Join

    结论：过滤条件写在`{subquery_where_condition}`、`{on_condition}`和`{where_condition}`不一定等价。

    对于左表的过滤条件，放在`{subquery_where_condition}`和`{where_condition}`是等价的。

    对于右表的过滤条件，放在`{subquery_where_condition}`和`{on_condition}`中是等价的，右表表达式不能放在`{where_condition}`中。

    LEFT ANTI Join的处理逻辑是对于左表的每一条记录，都去和右表进行匹配，如果右表所有的记录都没有匹配成功，则输出左表。同样由于只输出左表，所以JOIN后的Where条件中不能写右侧的过滤条件。LEFT ANTI JOIN常常用来实现not exists的语义。

    1.  第一种情况，子查询中过滤：

        ```
        SELECT A.*
        FROM
        (SELECT * FROM A WHERE ds='20180101') A
        LEFT ANTI JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key;
        ```

        过滤后，左侧有两条，右侧有两条，结果有1条：

        |a.key|a.ds|
        |:----|:---|
        |2|20180101|

    2.  第二种情况，JOIN 条件中过滤：

        ```
        SELECT A.*
        FROM A LEFT ANTI JOIN B
        ON a.key = b.key and A.ds='20180101' and B.ds='20180101';
        ```

        对于左侧的三条记录，只有第一条有满足ON条件的结果，所以输出剩余的两条记录：

        |a.key|a.ds|
        |:----|:---|
        |2|20180101|
        |2|20180102|

    3.  第三种情况，JOIN后的WHERE条件过滤：

        ```
        SELECT A.*
        FROM A LEFT ANTI JOIN
        (SELECT * FROM B WHERE ds='20180101') B
        ON a.key = b.key
        WHERE A.ds='20180101';
        ```

        左侧能通过ON条件的有两条：

        |a.key|a.ds|
        |:----|:---|
        |2|20180101|
        |2|20180102|

        此时对于这个结果再进行过滤`A.ds='20180101'`，结果为1条：

        |a.key|a.ds|
        |:----|:---|
        |2|20180101|

        可以看到，LEFT ANTI JOIN中，过滤条件放在JOIN ON条件中和前后的WHERE条件中，结果是不相同的。

        以上只是针对一个常用场景的几种不同的写法做的简单的测试，没有具体的推导过程，对于涉及到不等值表达式的场景会更加复杂，有兴趣的同学可以自己尝试推导一下。


## 线上状态 {#section_cwv_ctw_1fb .section}

上述结果都是在SQL标准语义模式下的推导结果。有的用户会发现在线上环境中，相同的语句得到的结果和预期并不相符，这是由于一些历史原因和兼容性的考虑。在OUTER JOIN的实现中，在project级别设置了一个flag，叫做`odps.sql.outerjoin.supports.filters`，如果这个设置为false，表示OUTER JOIN的ON条件不支持过滤条件，写在`{on_condition}`会被当做写在了`{subquery_where_condition}`，这是一个非标准的行为。有些用户在两个project中切换，发现相同的SQL在两个project中运行行为不一致，就是由这个引起的。

这里希望大家都能够按照标准的SQL语义来写SQL，这样才能保证后续SQL的可移植性。

查看project设置可以在 [http://adminconsole.odps.aliyun-inc.com/inn.view](http://adminconsole.odps.aliyun-inc.com/inn.view) 中`project管理`找到对应的project，查看属性。

## 总结 {#section_qxs_jtw_1fb .section}

过滤条件放在不同的位置语义可能大不相同，对于用户而言，如果只是进行过滤数据后再JOIN的操作，可以简要记住以下几点。

1.  INNER JOIN/LEFT SEMI JOIN 对于两侧的表达式可以随便写。
2.  LEFT JOIN/LEFT ANTI JOIN 左表的过滤条件要放到`{subquery_where_condition}`或者`{where_condition}`，右表的过滤条件要放到`{subquery_where_condition}`或者`{on_condition}`中。
3.  RIGHT JOIN和LEFT JOIN相反，右表的过滤条件要放到`{subquery_where_condition}`或者`{where_condition}`，左表的过滤条件要放到`{subquery_where_condition}`或者`{on_condition}。`
4.  FULL OUTER JOIN 只能放到`{subquery_where_condition}`中。

当然如果还是觉得规则比较复杂的话，最好的方法就是每次都把过滤条件写到子查询中。

