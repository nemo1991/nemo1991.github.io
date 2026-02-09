---
title: nginx-log-management-strategy
---

# 问题来源

nginx 的access.log 耗尽服务器磁盘空间，导致nginx服务异常，应用访问中断。

# 解决方案

鉴于 access.log 时效性，仅保留30天日志，压缩每日日志，并删除30天前日志。当然也可以通过其他手段将数据采集至其他平台，如通过filebeat采集至elk。在此仅记录服务器本地管理方式。

# 操作步骤

1. nginx的access.log按照日期命名，每日的access.log文件为一个独立文件。
```
# 在http配置节增加日期变量
map $time_iso8601 $logdate {
    '~^(?<ymd>\d{4}-\d{2}-\d{2})' $ymd;
    default                       'nodate';
}

# 使用变量定义路径
access_log /var/log/nginx/access-$logdate.log;
```

2. 添加cron定时任务，按照文件的修改时间进行归档，删除。

```
cron -e
# 每天凌晨把 1 天前的日志压缩成 .gz，把 30 天前的删除
0 1 * * * find /var/log/nginx/ -name "????-??-??.log" -mtime +1 -exec gzip {} \;
0 2 * * * find /var/log/nginx/ -name "????-??-??.log.gz" -mtime +30 -exec rm -f {} \;
```

# 其他
1. 可以考虑使用系统自带的logrotate方式进行日志切割归档操作。
```
/var/log/nginx/*.log {
    daily           # 每天切割一次
    missingok       # 如果日志文件丢失，不报错
    rotate 14       # 保留最近 14 天的日志
    compress        # 切割后压缩以节省空间
    delaycompress   # 延迟压缩到下一次切割
    notifempty      # 如果日志为空则不切割
    create 0640 www-data adm
    sharedscripts
    postrotate
        # 切割后通知 Nginx 重新打开日志文件
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```
2. 权限问题,nginx进程无法按照日期创建日志文件

rwx 4 read 2 write 1 x 执行 execute？

owner /group /other

rwxrwxr-x 775
rwxrwxrwx 777

```
# 查看nginx worker进行运行用户，通过ps命令或nginx.conf检查
ps aux | grep nginx | grep worker
# 检查目录权限,每一个父目录都有执行权限（x），才能进入该目录
ls -ld /var/log/nginx/
# 给予其他用户进入目录的权限
chmod +x /var/log
chmod +x /var/log/nginx

# 递归修改所有者
sudo chown -R nginx-user:nginx-user /var/log/nginx
sudo chmod -R 755 /var/log/nginx

# 赋予目录读写执行权限，文件读写权限，仅保留无需执行，通过上方命令已经赋予nginx执行用户权限
sudo find /var/log/nginx -type d -exec chmod 755 {} \;
sudo find /var/log/nginx -type f -exec chmod 644 {} \;


```

3. 在nginx运行时，删除日志文件，磁盘空间未释放。
```
# 在删除日志文件后，通过运行reload命令，磁盘空间并未释放。
nginx -s reload
# 通过losf查看，依然是nginx进程占据文件。
lsof +L1 | grep nginx | grep 'deleted'
# 查看nginx进程
ps -ef | grep nginx
# 会看到残留的“虚幻”进程，依旧在运行，正在关闭中而未立即关闭。
# 通过kill命令，停止虚幻进程，磁盘空间会释放
kill -9 pid 
```

4. 清理大文件注意事项
对于正在写入的大文件，千万不要用 rm. 不会破坏 Nginx 的文件句柄。
或者文件重命名？没试过
```
# 可以通过重定向命令
true > access.log
```

