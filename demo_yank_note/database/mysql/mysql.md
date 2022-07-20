# mysql

+ mysql{.mindmap}
    + SQL : structure query language
    + Docker start mysql V8.0
        + docker run -it --name=mysql_server -p 3306:3306 -e MYSQL_ROOT_PASSWORD=123456 -d mysqlImage
        + mysql 8.0远程访问需先创建用户
            + 创建用户：create user afu_snail@'%' identified with mysql_native_password by "123456";
            + 授权：grant all privileges on *.* to afu_snail@'%' with grant option;
            + 刷新权限： flush privileges;
            + 💌 8.0版加密规则必须是mysql_native_password才能被远程连接
    + [1]create database
        +  create database 库名；
        +  use 库名；
    + [2]create table 
        + 👨‍🍳创建表: create table 表名(field type limit) engine InnoDB charset utf8;    
        + desc 表名；查看表结构
        + 修改表增加字段🧚：alter table 表名 add column_name int limit;
        + 修改表删除字段🧚：alter table 表名 drop column xxxxx;
        + 修改某字段：alter table 表名 modify xxx int unsigned;// 字段改名 alter table 表名 change xxx1 xxx2;
    + 删除表
        + drop table if exists 表名；
    + configure file (find /etc -name my.conf)
    + 数据类型 
        + (int char decimal(5,2)/定位浮点数5位2位小数点 not null, default, unsigned, auto_increment, primary key) 
        + EX💌: text 文本类型
        + json json类型
        +  blob 二进制对象

    + note
        + 插入字符串和日期类型数据要加引号，否则会报错
        + 字段名和mysql保留字段冲突时，要加反引号区分
        + 不同编码汉字占用空间不同
            + unicode/ASCII 码中，一个英文字母(不分大小写)为一个字节，一个中文汉字为两个字节。
            + UTF-8 编码中，一个英文字为一个字节，一个中文为三个字节。
        

    + 💌
    + 💌
    + 💘
    + 💝
    + 💖
    + 💗
    + 💓
    + 💞
    + 💕
    + 💟

---
+ mysql CURD{.mindmap}
    +  插入
    + 修改
    + 删除
    + 查询
    + 💌
    + 💌
    + 💘
    + 💝
    + 💖
    + 💗
    + 💞
    + 💟





---




