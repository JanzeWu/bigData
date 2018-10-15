# Hive 行转列

将某个字段以数组或者列表的形式，按元素个数切分成相对应的行。

如将

```sql
select * from table_a;
```



| col_a | col_b | col_c |
| ----- | ----- | ----- |
| a     | b     | 1,2,3 |



转换成

```sql
select * from table_b;
```



| col_a | col_b | col_c_single |
| ----- | ----- | ------------ |
| a     | b     | 1            |
| a     | b     | 2            |
| a     | b     | 3            |

```sql
create table table_b as 
    select 
        col_a, col_b, col_c_single 
    from table_a 
        lateral view explode( split( col_c, ',' ) ) col_c_table as col_c_single;
```



# 列转行

将一列中多行转一行。如将上面table_b 转 table_a。

```sql
create table table_a as
	select 
		col_a, col_b, concat_ws(',', collect_set(col_c_single))
	from
		table_c
	group by col_a, col_b;
```

