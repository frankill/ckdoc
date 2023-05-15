添加配置到conf.d目录下backup_disk.xml,重启ch

``` xml
<clickhouse>
    <storage_configuration>
        <disks>
            <backups>
                <type>local</type>
                <path>/backups/</path>
            </backups>
        </disks>
    </storage_configuration>
    <backups>
        <allowed_disk>backups</allowed_disk>
        <allowed_path>/backups/</allowed_path>
    </backups>
</clickhouse>
```

执行sql测试

``` sql

-- 创建数据库
CREATE DATABASE tmp;

-- 创建表
CREATE TABLE tmp.test
(
  id String,
  pid UInt32,
  money Decimal32(2)
) ENGINE=TinyLog;

-- 插入数据
INSERT INTO tmp.test
SELECT randomPrintableASCII(10) AS id, randBinomial(10000, .6) AS pid, round(randUniform(10000, .4), 2) AS money
FROM numbers(100000);

-- 查询表中的记录数
SELECT COUNT(1) FROM tmp.test;

-- 备份表到磁盘（第一次备份）
BACKUP TABLE tmp.test TO DISK('backups', 'tmp_test.zip');

-- 插入数据
INSERT INTO tmp.test
SELECT randomPrintableASCII(10) AS id, randBinomial(10000, .6) AS pid, round(randUniform(10000, .4), 2) AS money
FROM numbers(500000);

-- 备份表到磁盘（第二次备份），并设置基于第一次备份的增量备份
BACKUP TABLE tmp.test TO DISK('backups', 'tmp_test_1.zip') SETTINGS base_backup = Disk('backups', 'tmp_test.zip');

-- 备份整个数据库到磁盘
BACKUP DATABASE tmp TO DISK('backups', 'tmp_database.zip');

-- 查询系统中的备份列表
SELECT * FROM system.backups;

-- 删除表
DROP TABLE tmp.test;

-- 从磁盘中还原表（第二次备份,全量还原）
RESTORE TABLE tmp.test FROM DISK('backups', 'tmp_test_1.zip');

-- 删除表
DROP TABLE tmp.test;

-- 从磁盘中还原表（第一次备份）
RESTORE TABLE tmp.test FROM DISK('backups', 'tmp_test.zip');

-- 删除数据库
DROP DATABASE tmp;

-- 从磁盘中还原整个数据库
RESTORE DATABASE tmp FROM DISK('backups', 'tmp_database.zip');

```
