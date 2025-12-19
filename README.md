项目目标

编写两个Shell脚本：

1. backup.sh: 实现关键数据和配置文件的定时备份。
2. monitor_nginx.sh: 对Nginx服务的运行状态进行监控，异常时尝试恢复并告警。

· 操作系统： CentOS 7/8
· 核心工具： Shell脚本，crontab，mailx (或外部邮件API)，ps, grep, systemctl
· 目标服务： Nginx, MySQL (用于备份演示)

第一部分：自动化备份脚本 backup.sh

这个脚本的目的是定期备份两个关键内容：
1. 网站数据： 例如 /usr/share/nginx/html 目录。
2. 数据库： 例如MySQL中的 wordpress 数据库。

详细步骤：

1. 创建脚本文件并赋予执行权限
   touch /opt/backup.sh
   mkdir -p /opt/backups
   chmod +x /opt/backup.sh
  
3. 编写脚本内容 vim /opt/backup.sh
   #!/bin/bash
   
   # 定义变量
   BACKUP_DIR="/opt/backups"
   DATE=$(date +%Y%m%d_%H%M%S) # 备份日期，精确到秒，避免重名
   MYSQL_USER="root"
   MYSQL_PASSWORD="your_mysql_root_password" # 请替换为你的真实密码
   MYSQL_DATABASE="wordpress"
   
   # 1. 备份网站文件
   echo "$(date)：开始备份网站文件..." >> $BACKUP_DIR/backup.log
   tar -czf $BACKUP_DIR/website_backup_$DATE.tar.gz /usr/share/nginx/html/ 2>> $BACKUP_DIR/backup.log
   
   if [ $? -eq 0 ]; then
       echo "$(date)：网站文件备份成功！" >> $BACKUP_DIR/backup.log
   else
       echo "$(date)：【错误】网站文件备份失败！" >> $BACKUP_DIR/backup.log
       # 这里可以添加邮件告警逻辑
   fi
   
   # 2. 备份MySQL数据库
   echo "$(date)：开始备份MySQL数据库..." >> $BACKUP_DIR/backup.log
   mysqldump -u$MYSQL_USER -p$MYSQL_PASSWORD $MYSQL_DATABASE > $BACKUP_DIR/db_backup_$DATE.sql 2>> $BACKUP_DIR/backup.log
   
   if [ $? -eq 0 ]; then
       echo "$(date)：MySQL数据库备份成功！" >> $BACKUP_DIR/backup.log
       # 压缩SQL文件以节省空间
       gzip $BACKUP_DIR/db_backup_$DATE.sql
   else
       echo "$(date)：【错误】MySQL数据库备份失败！" >> $BACKUP_DIR/backup.log
       # 这里可以添加邮件告警逻辑
   fi
   
   # 3. 清理7天前的备份文件（可选，防止磁盘占满）
   find $BACKUP_DIR -name "*.gz" -type f -mtime +7 -delete
   find $BACKUP_DIR -name "*.log" -type f -mtime +30 -delete
   
   echo "$(date)：本次备份任务全部完成。" >> $BACKUP_DIR/backup.log

3. 配置定时任务 crontab
   # 编辑当前用户的cron任务
   crontab -e
   · 在末尾添加一行，表示每天凌晨2点执行备份脚本：
   cron
   # 分 时 日 月 周 要执行的命令
   0 2 * * * /opt/backup.sh > /dev/null 2>&1
   · 保存并退出。现在脚本就会每天自动运行了。
第二部分：服务监控脚本 monitor_nginx.sh

这个脚本的目的是周期性地检查Nginx是否在运行，如果不在，则尝试重启并通知管理员。

详细步骤：

1. 创建脚本文件并赋予执行权限
   touch /opt/monitor_nginx.sh
   touch /opt/service_monitor.log
   chmod +x /opt/monitor_nginx.sh
   
3. 配置邮件发送功能（告警前提）
   · 方法A：使用 mailx 配置系统自带的Sendmail/Postfix（较复杂）。
   · 方法B（推荐-模拟）： 我们用一个写入日志文件的动作来模拟“发送邮件”，可以说明“此处可以集成邮件API或钉钉/webhook”。
   · 为了演示，我们这里安装一个简单的 mailx：
   yum install mailx -y
   · 一个简单的配置示例（需要你有一个可用的SMTP服务器，如QQ邮箱）：
   # 编辑 /etc/mail.rc，在末尾添加：
   set from=your_email@qq.com
   set smtp=smtp.qq.com
   set smtp-auth-user=your_email@qq.com
   set smtp-auth-password=你的授权码 # 不是QQ密码，是SMTP授权码
   set smtp-auth=login
   set ssl-verify=ignore
   set nss-config-dir=/etc/pki/nssdb/
  
4. 编写监控脚本 vim /opt/monitor_nginx.sh
   #!/bin/bash
   
   # 定义变量
   SERVICE="nginx" # 要监控的服务名
   LOG_FILE="/opt/service_monitor.log"
   ADMIN_EMAIL="admin@yourcompany.com" # 管理员邮箱
   
   # 检查服务进程是否存在
   if ! pgrep -x "$SERVICE" > /dev/null
   then
       # 如果进程不存在，记录日志并告警
       echo "$(date)：【危机】服务 $SERVICE 已停止！正在尝试重启..." >> $LOG_FILE
       
       # 尝试重启服务
       systemctl restart $SERVICE
       
       # 等待几秒再次检查是否重启成功
       sleep 5
       
       if pgrep -x "$SERVICE" > /dev/null
       then
           RESTART_MSG="$(date)：服务 $SERVICE 重启成功。"
           echo $RESTART_MSG >> $LOG_FILE
           # 发送成功通知（可选）
           echo "$RESTART_MSG" | mail -s "服务恢复通知：$SERVICE" $ADMIN_EMAIL
       else
           ERROR_MSG="$(date)：【严重错误】服务 $SERVICE 重启失败，需要手动干预！"
           echo $ERROR_MSG >> $LOG_FILE
           # 发送错误告警
           echo "$ERROR_MSG" | mail -s "紧急告警：$SERVICE 服务异常" $ADMIN_EMAIL
       fi
   else
       # 服务正常运行，可以记录一条正常日志（为避免日志过多，通常不记录）
       echo "$(date)：服务 $SERVICE 运行正常。" >> $LOG_FILE
   fi
   4. 配置定时任务，实现每分钟检查一次
   crontab -e
   # 每分钟检查一次Nginx服务
   * * * * *  /opt/monitor_nginx.sh > /dev/null 2>&1









   
