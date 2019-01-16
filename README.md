# Zabbix Template for MySQL with replication

- Based on the official Zabbix MySQL template
- Uses Zabbix agent user parameters (passive checks)

## Configuration
1. Install Zabbix agent (https://www.zabbix.com/download)

2. Backup and remove existing user parameters in the file
```
> /etc/zabbix/zabbix_agentd.d/userparameter_mysql.conf
```

3. Provide MySQL credentials and set correct permissions (example for Ubuntu)
```
chmod 640 /etc/mysql/debian.cnf
chown root:zabbix /etc/mysql/debian.cnf
```

4. Add required user user parameters
```
# Flexible parameter to grab global variables. On the frontend side, use keys like mysql.status[Com_insert].
# Key syntax is mysql.status[variable].
UserParameter=mysql.status[*],echo "show global status where Variable_name='$1';" | HOME=/var/lib/zabbix mysql -N | awk '{print $$2}'

# Flexible parameter to determine database or table size. On the frontend side, use keys like mysql.size[zabbix,history,data].
# Key syntax is mysql.size[<database>,<table>,<type>].
# Database may be a database name or "all". Default is "all".
# Table may be a table name or "all". Default is "all".
# Type may be "data", "index", "free" or "both". Both is a sum of data and index. Default is "both".
# Database is mandatory if a table is specified. Type may be specified always.
# Returns value in bytes.
# 'sum' on data_length or index_length alone needed when we are getting this information for whole database instead of a single table
UserParameter=mysql.size[*],bash -c 'echo "select sum($(case "$3" in both|"") echo "data_length+index_length";; data|index) echo "$3_length";; free) echo "data_free";; esac)) from information_schema.tables$([[ "$1" = "all" || ! "$1" ]] || echo " where table_schema=\"$1\"")$([[ "$2" = "all" || ! "$2" ]] || echo "and table_name=\"$2\"");" | /usr/bin/mysql --defaults-extra-file=/etc/mysql/debian.cnf -N'

UserParameter=mysql.ping,mysqladmin --defaults-extra-file=/etc/mysql/debian.cnf ping | grep -c alive
UserParameter=mysql.version,mysql --defaults-extra-file=/etc/mysql/debian.cnf -V

UserParameter=mysql.extended_status[*],/usr/bin/mysqladmin --defaults-extra-file=/etc/mysql/debian.cnf extended-status variables |awk {'print $$2"| "$$4'} | grep "$1|" | awk {'print $$2'}

UserParameter=mysql.slave_lagging,/usr/bin/mysql --defaults-extra-file=/etc/mysql/debian.cnf -Bse "show slave status\\G" | grep Seconds_Behind_Master | awk '{print $$2}' | sed -e 's/^NULL$/-1/; s/![0-9]+/-1/' | awk '{if($2 ~ /d/) {print 100} else {print $2}}'

UserParameter=mysql.slave_status[*],/usr/bin/mysql --defaults-extra-file=/etc/mysql/debian.cnf -e "show slave status\\G"|grep -i $1 | awk '{print $$2}'
```
