**一、逻辑图**

![](https://agam-blog-image.oss-cn-hangzhou.aliyuncs.com/db_auto_increase.png)

**二、说明**

**参数**

* auto_increment_increment // 步长
* auto_increment_offset // 起点

Tip：上述参数可以进行session级别与全局级别两种方式设置

**表**

CREATE TABLE id_generator (
    seqid bigint(20) unsigned NOT NULL auto_increment,
    seqname varchar(50) NOT NULL default '',
    PRIMARY KEY (seqid),
    UNIQUE KEY seqname (seqname)
) ENGINE=INNODB;

**存储过程**

begin;
    REPLACE INTO id_generator(seqname) VALUES ('lottery_order');
    SELECT LAST_INSERT_ID();
commit;
