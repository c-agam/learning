**一、逻辑图**

![](https://agam-blog-image.oss-cn-hangzhou.aliyuncs.com/db_batch_id.png)

**优点**
* 保证了ID生成的绝对递增有序；
* 大大的降低了数据库的压力，ID生成可以做到每秒生成几万几十万个。

**缺点**
* 服务是单点；
* 如果服务挂了，服务重启起来之后，继续生成ID可能会不连续，中间出现空洞（服务内存是保存着0,1,2,3,4,5，数据库中max-id是5，分配到3时，服务重启了，下次会从6开始分配，4和5就成了空洞，不过这个问题也不大）；
* 虽然每秒可以生成几万几十万个ID，但毕竟还是有性能上限，无法进行水平扩展。

**改进方案**

单点服务的常用高可用优化方案是“备用服务”，也叫“影子服务”，所以我们能用以下方法优化上述缺点：对外提供的服务是主服务，有一个影子服务时刻处于备用状态，当主服务挂了的时候影子服务顶上。这个切换的过程对调用方是透明的，可以自动完成，常用的技术是vip+keepalived。

**二、附录**
```
// 表结构
CREATE TABLE id_generator (
    seqname varchar(50) NOT NULL default '',
    seqid bigint(20) unsigned NOT NULL,
    PRIMARY KEY (seqname)
) ENGINE=INNODB;

// Java 代码
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.sql.DataSource;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.sql.Statement;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 分布式应用全局序列号申请,通过数据库行锁(悲观锁)来实现从
 * 数据库批量申请序列号,该种方案当应用重启时会浪费一定的序列号,
 * 浪费的粒度取决于ID_BATCH_SZIE常量的值
 * @author zul
 * @since 2018-01-31
 * @version 0.0.1
 */
@Service
public class SeqGenerator implements ISeqGenerator {
    /**
     * 批申请量
     */
    private static final int ID_BATCH_SZIE = 10;
    /**
     * 序列号本地缓存
     */
    private static final Map<SeqType,IdRange> cache = new ConcurrentHashMap<>();

    @Autowired
    private DataSource dataSource;

    @Override
    public Long getSeq(SeqType seqType) {
        synchronized (seqType)
        {
            IdRange idRange = cache.get(seqType);
            if(idRange != null)
            {
                long id = idRange.fetchNext(1);
                if(id != -1)
                {
                    return id;
                }
            }

            idRange = new IdRange(loadNextBatch(seqType), ID_BATCH_SZIE);
            cache.put(seqType, idRange);
            return idRange.fetchNext(1);
        }
    }

    /**
     * 通过行锁结合事务实现分布式应用批量申请序列号
     * @param seqType
     * @return
     */
    private long loadNextBatch(SeqType seqType)
    {
        long currentSeqId = 0;
        Connection c = null;

        try {
            c = dataSource.getConnection();
            c.setAutoCommit(false);
            Statement s = c.createStatement();
            s.setQueryTimeout(50);

            boolean exists = true;
            ResultSet rs = s.executeQuery("SELECT seqid FROM id_generator WHERE seqname='" + seqType.name() + "' FOR UPDATE");
            if (rs.first()) {
                currentSeqId = rs.getLong(1);
            } else {
                exists = false;
            }
            rs.close();

            if (exists) {
                s.executeUpdate("UPDATE id_generator SET seqid=" + (currentSeqId + ID_BATCH_SZIE) + " WHERE seqname='" + seqType.name() + "'");
            } else {
                s.executeUpdate("INSERT INTO id_generator(seqname,seqid) VALUES('" + seqType.name() + "'," + 1000000000 + ")");
            }

            c.commit();
            s.close();
        } catch (Exception e) {
            if (c != null) {
                try {
                    c.rollback();
                } catch (SQLException e1) {
                    e1.printStackTrace();
                }
            }
            throw new IllegalStateException(e);
        } finally {
            if (c != null) {
                try {
                    c.close();
                } catch (SQLException e) {
                    e.printStackTrace();
                }
            }
        }
        return currentSeqId + 1;
    }

    /**
     * 已经申请序列对象
     */
    private static class IdRange {
        private long current = 0;
        private long max = 0;

        public IdRange(long start, long total)
        {
            update(start, total);
        }

        public void update(long start, long total)
        {
            this.current = start;
            this.max = start + total;
        }

        public long fetchNext(int count) {
            long id = -1;

            if (this.current + count <= max) {
                id = this.current;
                this.current += count;
            }
            return id;
        }
    }
}
```
