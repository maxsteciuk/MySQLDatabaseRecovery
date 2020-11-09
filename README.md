# MySQL Database Recovery
Instructions to recover a corrupted MySQL database

In case a MySQL database is corrupted such as only its internal state is corrupted but data itself is intact it may still be possible to recover a database. The symptom of such corruption is that MySQL has failed to load database tables due to corrupted or missing indexes. The sample error logging when such corruption has happened looks as follows:
```
2020-03-08T19:52:25.634251Z 0 [ERROR] [FATAL] InnoDB: Table flags are 0 in the data dictionary but the flags in file /var/db/mysql/ibdata1 are 0x4800!
0x7af00e <handle_fatal_signal+0x2ae> at /usr/local/libexec/mysqld
2020-03-08 15:52:25 0x804418000  InnoDB: Assertion failure in thread 34431139840 in file ut0ut.cc line 918
InnoDB: We intentionally generate a memory trap.
InnoDB: Submit a detailed bug report to http://bugs.mysql.com.
InnoDB: If you get repeated assertion failures or crashes, even
InnoDB: immediately after the mysqld startup, there may be
InnoDB: corruption in the InnoDB tablespace. Please refer to
InnoDB: http://dev.mysql.com/doc/refman/5.7/en/forcing-innodb-recovery.html
InnoDB: about forcing recovery
```
To  achieve the database recovery in this case a set of utilities from the [Undrop-for-innodb](https://github.com/maxsteciuk/undrop-for-innodb) repository is used.

**CAUTION Please ensure backing up a MySQL database directory before proceeding with the below steps**

# Steps to recover MySQL tables in ibd format
1. Copy **ibdata1** from **/var/db/mysql/**  to a working directory for the recovery procedure and run the following command to parse **ibdata1** file:
```
./stream_parser -f ibdata1
```
2. Parse **SYS_TABLES** from **ibdata1** pages:
```
./c_parser -Uf pages-ibdata1/FIL_PAGE_INDEX/0000000000000001.page -p mysqldb.bak/ -t dictionary/SYS_TABLES.sql 2> SYS_TABLES .sql | sort -nk 4,5 > dumps/mysqldb_recovered/SYSTABLES
```

3. Parse **SYS_INDEXES** from **ibdata1** pages:  
```
./c_parser -Uf pages-ibdata1/FIL_PAGE_INDEX/0000000000000003.page -p mysqldb.bak/ -t dictionary/SYS_INDEXES.sql   2> SYS_INDEXES.sql | sort -nk 4,5 > dumps/mysqldb_recovered/SYSINDEX
```

4. Copy **<table_name>.ibd** from **/var/db/mysql/<database_name>**  to a working directory for the recovery procedure and parse a MySQL table **ibd** file:
```
./stream_parser -f <table_name>.ibd
```

5. Parse MySQL table from its corresponding **.ibd** parsed page index given its **create table** schema. The page index number can be found by looking up **PRIMARY** column value in the parsed **dumps/SYSTABLE** and correlating it with a corresponding row in the parsed **dumps/SYSINDEX**. Then using that information run the following command:
```
./c_parser -6f ./pages-<table_name>.ibd/FIL_PAGE_INDEX/0000000000000222.page -t create_<table_name>.sql > dumps/<table_name>.tsv 2> load_cmd.sql
```

6. Finally load a recovered table into MySQL: 
```
mysql --loose-local-infile=1 -u root -p -D <database_name> < load_cmd.sq
```
## The steps are to be repeated for each corrupted table

# Steps to recover MySQL tables in ISAM format
## Usually these steps apply to a WordPress database

1. Go to the following directory **/var/db/mysql/<database_name>** and run the following command:
```
myisamchk <table_name>
```
2. If there is an error for a table from a previous command, then run the followinng command:
```
myisamchk -q -r  <table_name>
```
3. If **.frm** file is lost  copy **<table_name>.frm** from a currently working MySQL database
