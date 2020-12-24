
在日常mysql使用过程中，通常会涉及到mysql库的迁移，库表数据的同步导入/导出。或者是将一些线上库表数据导出到本地，都会涉及到常见的mysql数据处理流程，那么就需要总结一下常见的mysql针对数据，库表，结构的数据的导入导出方式。


#### 将线上库表数据导出文本

 需要先登陆数据库服务器，通过mysql命令指定参数添加sql查询结果导出到当前服务器路径。后面通过`sc,zc`进行传输。详情参考另外一篇文章:[mac使用sz,rz远程传输文件](https://www.xssor2600.site/post/mac_iterm2_rz_use/)

 ```sql
mysql -h 10.224.40.35 -uuser -p'password'  -Ddatabase --default-character-set=utf8  -e"select column1,column2... where payee_logon_id in('13960225506','15905957688','15905023789','13641490558','15260388243','17305081757')" >tt.log

// 另外一种方式
echo "select uid,\`order\`,money,country_code from user_order_record_new_202006 where create_time >= '2020-06-18 00:00:00' and manner_id = 3 and client = 1 and agree =1" | mysql -h xxxxx -uroot -Ddemo -p'123456' --default-character-set=utf8  > /tmp/ios_recharge_18.xls;

//
mysqldump -uinke_db_user -h10.100.130.39 -p --default-character-set=utf8 --single-transaction --master-data --where='type = 1 and money > 1 and money < 9998' inke_payment rule_charge > TableConditon.sql

 ```



有时候想要做库表级别的查询？

```sql
select * from information_schema.tables where TABLE_SCHEMA='demo' and table_rows>0 and TABLE_NAME like 'gold%' limit 10\G;
```



---



#### 库表迁移

- 有时候需要将一个库里面的所有表结构进行复制，但是不需要表数据，如何处理？</br>

  **方式1:**
        通过指定查询sql将要清空的表sql命令拼接列出，后面可以批量执行(复制truncate语句到mysql命令行执行，可以一次复制多条执行)	

  ```sql
  mysql -hlocalhost -uroot -px1234567 -N -s information_schema -e "SELECT CONCAT('TRUNCATE TABLE ',table_schema,'.',TABLE_NAME,';') FROM TABLES WHERE TABLE_SCHEMA='test'"
  mysql: [Warning] Using a password on the command line interface can be insecure.
  TRUNCATE TABLE test.demo;   // 需要执行的sql脚本
  ...
  ```

  ​		  **方式2:**
  ​         通过mysqldump命令先将原来库内表结构导出，然后直接drop掉库，然后新建库后导入表结构通过`source`命令将sql文件导入数据库。

  ```sql
  mysqldump -h localhost -uroot -proot -d databaseT > emtpy_tables.sql
  
  mysql> drop database databaseT;
  Query OK, 62 rows affected, 1 warning (0.28 sec)
  
  mysql> CREATE DATABASE `databaseT` CHARACTER SET utf8 COLLATE utf8_general_ci;
  Query OK, 1 row affected (0.00 sec)
  
  mysql> use databaseT;
  Database changed
  mysql> source /tmp/empty_tables.sql
  ```

  

  针对`mysqldump`常见的对库表的使用方式：

  1. 导出指定表的数据

     ```sql
     mysqldump -t database -u username -ppassword --tables table_name1 table_name2 table_name3 >D:\db_script.sql
     ```

     

  2. 导出指定表的结构

     ```sql
     mysqldump -d database -u username -ppassword --tables table_name1 table_name2 table_name3>D:\db_script.sql
     ```

     

  3. 导出表的数据及结构

     ```sql
     mysqldump  database -u username -ppassword --tables table_name1 table_name2 table_name3>D:\db_script.sql
     ```

     

  4. 若 数据中 ，某些表除外，其余表都需导出

     可以使用`--ignore-table`参数指定跳过导出的表结构数据。

     ```sql
     mysqldump -h ip -u username -ppassword --default-character-set=utf8 --database database_name --ignore-table=database_name.table_name1 > D:\db_script.sql
     ```



- 如何将*.sql语句导入数据库

  从数据库中导出数据，参考上面内容。那么如何将sql语句导入数据库？

  ```sql
  // 方式1:（source） 首先建空数据库/导入数据库
  mysql>create database abc;
  mysql>use abc;
  mysql>source /home/abc/abc.sql;
  
  // 方式2: 直接命令行导入
  mysql -u用户名 -p密码 数据库名 < 数据库名.sql
  #mysql -uabc_f -p abc < abc.sql
  # mysql -h 127.0.0.1 -uroot -proot payment < payment_base_table.sql
  
  ```

  


