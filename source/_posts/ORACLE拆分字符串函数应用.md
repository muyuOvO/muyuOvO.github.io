---
title: ORACLE拆分字符串函数应用
date: 2023-3-23 16:32:00
updated: 2023-3-23 15:00:00
categories: 
  - ORACLE
---
## ORACLE中拆分字符串中的键值对应用
> 业务场景：设备A的多个端口数据通过键值对的方式传入，需要对其不同端口的指标值进行汇聚操作，因为历史遗留问题+使用场景较少JAVA入库不想写对应模板拆分 只能用数据库来操作啦🤣🤣


涉及的两个表：入库表：DEMO1，拆分数据后的中间表 DEMO1_MID
表结构和样例数据如下：
DEMO1:
|KEY_P|KPI_1|KPI_2|
|:---:|:---:|:---:|
|A|INDEX_1,10;INDEX_2,20;|INDEX_3,30;|
|B|INDEX_4,40;|INDEX_5,50;INDEX_6,60;|

DEMO1_MID:
|KEY_P|KEY_|VALUE|
|:---:|:---:|:---:|


### 1.首先要把字段进行拆分

拆分函数：split_key_value_pairs
```sql
CREATE OR REPLACE FUNCTION split_key_value_pairs(
    input_string IN VARCHAR2,
    delimiter IN VARCHAR2 DEFAULT ';',
    key_value_delimiter IN VARCHAR2 DEFAULT ','
) RETURN SYS_REFCURSOR
AS
    output_cursor SYS_REFCURSOR;
BEGIN
    OPEN output_cursor FOR
        SELECT SUBSTR(input_string, 1, INSTR(input_string, key_value_delimiter)-1) AS key,
               SUBSTR(input_string, INSTR(input_string, key_value_delimiter)+1) AS value
          FROM (
               SELECT TRIM(REGEXP_SUBSTR(input_string, '[^' || delimiter || ']+', 1, LEVEL)) AS input_string
                 FROM DUAL
           CONNECT BY LEVEL <= REGEXP_COUNT(input_string, '[^' || delimiter || ']+')
               )
         WHERE input_string IS NOT NULL;
    RETURN output_cursor;
END;
```
利用这个函数可以返回一个'SYS_REFCURSOR'类型的游标，其中包含两列，一列为键，一列为值，游标可以使用 SELECT 语句来读取。



### 2.在存储过程或者命令行用游标对其进行读取
基本上都是在存储过程了，命令行就用来测试测试效果。
```sql
DECLARE
  output_cur SYS_REFCURSOR;
  key_value_pair VARCHAR2(30000) ;
  output_key VARCHAR2(4000);
  output_value VARCHAR2(4000);
  v_key_p VARCHAR2(4000);
BEGIN
  FOR cur IN 
      (SELECT  KEY_P,KPI_1,KPI_2 FROM DEMO1 --WHERE  可能有其他的限制条件
      )   
    LOOP
      key_value_pair := cur.KPI_1 || cur.KPI_2;
      v_key_p := cur.KEY_P;
      output_cur := split_key_value_pairs(key_value_pair);
        LOOP
            FETCH output_cur INTO output_key, output_value;
            EXIT WHEN output_cur%NOTFOUND;
            INSERT INTO DEMO1_MID
            VALUES 
            (
            v_key_p,
            output_key,
            output_value
            );
        END LOOP;
        CLOSE output_cur;
    END LOOP;
END;
/

```
利用了双循环，先在DEMO1里面一次扫描每一行，将KPI_1,KPI_2进行拼接利用函数拆分，写进中间表DEMO1_MID
最后的数据结果就是酱紫：
|KEY_P|KEY_|VALUE|
|:---:|:---:|:---:|
|A|INDEX_1|10|
|A|INDEX_2|20|
|A|INDEX_3|30|
|B|INDEX_4|40|
|B|INDEX_5|50|
|B|INDEX_6|60|


这样，设备以及各个端口数据就可以进行下一步操作了👀