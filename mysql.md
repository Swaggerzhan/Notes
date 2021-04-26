# mysql

### 1. mysql数据库的一些sql语句


* 1. 创建表结构
    
    ```sql
    /* 创建表，建议将其保存在.sql文件中再使用 mysql < 表.sql直接运行。 */
    CREATE TABLE member(
        member_id INT UNSIGNED NOT NULL AUTO_INCREMENT,
        PRIMARY KEY (member_id), /* 主键，默认必须要一个主键(索引) */
        last_name   VARCHAR(20) NOT NULL, /* 不可为空 */
        first_name  VARCHAR(20) NOT NULL,
        suffix      VARCHAR(5) NULL, /* 默认为空 */
        expiration  DATA NULL,
        email       VARCHAR(100) NULL,
        street      VARCHAR(50) NULL,
        city        VARCHAR(50) NULL,
        state       VARCHAR(2) NULL,
        zip         VARCHAR(10) NULL,
        phone       VARCHAR(20) NULL,
        intersets   VARCHAR(255) NULL
    );
    /* 使用DESCRIBE语句+表名可以显示表字段，简写DESC + 表名 */
    ```
* 2. WHERE
    where是条件语句，使用where可以过滤一些消息，过滤显示如 __SELECT id,name FROM 表名 WHERE id=1;__ 可以将条件定位id=1，并且将其显示出来。