# RDS

## RDS slow queries or locked

```bash
#download the RDS slow log from aws
aws rds download-db-log-file-portion --db-instance-identifier <RDS_NAME> --log-file-name slowquery/mysql-slowquery.log >> ~/tmp/mysql-slowquery.log

#analyse the downloaded logs
pt-query-digest ~/tmp/mysql-slowquery.log
```

Please note that RDS splits up the slow query log in separate files per hour: `slowquery/mysql-slowquery.log`, `slowquery/mysql-slowquery.log.0`, `slowquery/mysql-slowquery.log.1` .. , `slowquery/mysql-slowquery.log.22`.

`pt-query-digest` tries to analyze these slow query logs to digest meaningfull data from it. It does this by looking at a combination of different factors like query time, amount of occurences etc. To get the most usefull digest it's recommended to feed it a large enough dataset, by combining several log files together:

```bash
pt-query-digest ~/tmp/mysql-slowquery.log* > digest.log
```
