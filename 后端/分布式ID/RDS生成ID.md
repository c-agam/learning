![](https://agam-blog-image.oss-cn-hangzhou.aliyuncs.com/db_auto_increase.png)

```
// 步长
auto_increment_increment 
// 起点
auto_increment_offset

// Tip：上述参数可以进行session级别与全局级别两种方式设置

// 序列表
CREATE TABLE id_generator (
    seqid bigint(20) unsigned NOT NULL auto_increment,
    seqname varchar(50) NOT NULL default '',
    PRIMARY KEY (seqid),
    UNIQUE KEY seqname (seqname)
) ENGINE=INNODB;

// 存储过程
begin;
    REPLACE INTO id_generator(seqname) VALUES ('lottery_order');
    SELECT LAST_INSERT_ID();
commit;
```
