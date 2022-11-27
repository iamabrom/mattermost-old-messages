This guide will help you delete old messages and attachments from your Mattermost Open Source edition server.

This procedure is very similar to what the [data retention](https://docs.mattermost.com/administration/data-retention.html) option of Mattermost `Enterprise Edition E20` does.

:warning: Run this at your own risk! I strongly suggest making a backup of your Mattermost directories and MySQL database before running these commands :warning:

# Generate a timestamp
This will be the limit date for the messages to keep (messages before this date will be deleted).
Use https://www.epochconverter.com/ or Unix commands to generate the timestamp for your date.

- For example: `1527811200` corresponds to the first of june 2018.
- In Mattermost database timestamps are stored with sub seconds accuracy so you need to multiply this timestamp by 1000 for later use.

In this example we will use `1527811200000`, tweak the commands with your own timestamp.

## List old messages / attachments
Open the MySQL prompt on your server and issue these queries

```sql
USE mattermost;
SELECT * FROM FileInfo WHERE CreateAt < 1527811200000 LIMIT 10;
SELECT * FROM Posts WHERE CreateAt < 1527811200000 LIMIT 10;
```
You should see information about the files / messages, this only lists 10 results.

If you get an empty set make sure you have the right timestamp (multiplied by 1000)

## Delete old messages / attachments

### MySQL configuration
Edit `/etc/mysql/mysql.conf.d/mysqld.cnf` and add at the end

```bash
secure_file_priv=""
```

Restart MySQL

```bash
sudo service mysql restart
```

This will allow us to dump the results of a query into a file on the disk.

### Get a list of the attachments
- Tweak the user and password corresponding to your credentials.
- This will create a temporary directory with a file inside containing the path of each attachment that is to be removed

```bash
#!/bin/bash
sudo mkdir /tmp/mysqldump/
sudo chown -R mysql:mysql /tmp/mysqldump/
sudo rm /tmp/mysqldump/*
mysql -u user -p'password' -D mattermost -e "SELECT Path FROM FileInfo \
WHERE CreateAt < 1527811200000 \
INTO OUTFILE '/tmp/mysqldump/uploaded_files.txt' \
FIELDS TERMINATED BY ',' \
ENCLOSED BY '' \
LINES TERMINATED BY '\n';"
```

### Script to delete the attachments
Create an executable script with the following content, tweak the `/opt/mattermost/data` with your installation directory of Mattermost.

```bash
#!/bin/bash
input="/tmp/mysqldump/uploaded_files.txt"
while IFS= read -r line
do
  rm "/opt/mattermost/data/$line"
done < "$input"
```

Run this script as root to delete the attachments files.

You can now delete the list of files

```bash
sudo rm -Rf /tmp/mysqldump/
```

### Delete the database entries
This will delete all the messages/attachments in the database

```sql
mysql -u user -p'password' -D mattermost -e "DELETE FROM FileInfo WHERE CreateAt < 1527811200000;"
mysql -u user -p'password' -D mattermost -e "DELETE FROM Posts WHERE CreateAt < 1527811200000;"
```

### Configuration MySQL
We can now revert the  MySQL configuration at `/etc/mysql/mysql.conf.d/mysqld.cnf`, comment

```bash
#secure_file_priv=""
```

Restart MySQL
```bash
sudo service mysql restart
```
