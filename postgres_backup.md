# How to Automatically Back Up PostgreSQL Databases with Object Storage

## Introduction

Regular database backups allow you to restore your application in case of contingencies, such as a server crash. Furthermore, you are responsible for safekeeping of user data associated with your application. Thus, on production servers, it is necessary to routinely back up the database. Storing the backups remotely using Vultr Object Storage means you do not need to provision and manage separate servers to maintain redundant backups. Using Vultr Object Storage for backups also gives you flexibility in storage space. The process of backing up the database to the cloud should be reliable and free of human intervention. Thus, it is advisable to automate the process.  

At a high level, the back up process consists of these steps:

 * Make a dump of the database using the `pg_dump` utility. This dump consists of SQL commands. Executing these SQL commands recreates the tables and the data of the database.
 * Back up the dump to a cloud store, such as Vultr Object Storage.

This guide shows how to automatically periodically back up PostgreSQL databases to Vultr Object Storage. 

## Prerequisites

To follow the instructions in this guide, it is necessary to: 

 * [Install PostgreSQL on the server.](https://docs.vultr.com/how-to-install-configure-backup-and-restore-postgresql-on-ubuntu-20-04-lts)
 * [Create an instance of Vultr Object Storage](https://docs.vultr.com/vultr-object-storage).
 * Log into the server as a [non-root sudo user](https://docs.vultr.com/how-to-use-sudo-on-a-vultr-cloud-server#create-a-sudo-user-on-debian-&-ubuntu). This guide assumes you log in with the username `pguser`.

### Install and Configure s3cmd

`s3cmd` is a command line application used to programmatically access object storage platforms, such as Vultr Object Storage, over the S3 protocol. Install `s3cmd`:

```CONSOLE
$ sudo apt install -y s3cmd
```

The above command installs the `s3cmd` application.

Before using Vultr Object Storage with `s3cmd`, it is necessary to configure `s3cmd` for your Object Storage instance. The article about [using s3cmd with Vultr Object Storage](https://docs.vultr.com/how-to-use-s3cmd-with-vultr-object-storage#set-default-configuration) shows how to do that. 

### Create Temporary Directory

Create a working directory for temporarily storing database dumps before backing up to Vultr Object Storage. After backing up to the cloud, the local database dumps are deleted. This helps save storage space on the server. 

Create the working directory `pgtemp` under the `/tmp` path:

```CONSOLE
$ mkdir /tmp/pgtemp/
```

Switch to this temporary directory:

```CONSOLE
$ cd /tmp/pgtemp/
```

### Create PostgreSQL User

Add your current username as a PostgreSQL superuser:

```CONSOLE
$ sudo -u postgres createuser --superuser $USER
```

Also create a default database for your current user id. When you log into PostgreSQL, you connect to this database by default.

```CONSOLE
$ sudo -u postgres createdb $USER
```

The example commands in this guide connect to PostgreSQL with your Linux username as the database user.

### Create a Test Database

The illustration commands in this guide are based on a test database called `my_db`. This section shows how to create the test database from the `psql` command line. Start a new PSQL interface:

```CONSOLE
$ psql
```
    
To test the example commands, create a new database 'my_db':

```PSQL
=# CREATE DATABASE my_db;
```

Connect to the database `my_db`:

```PSQL
=# \c my_db
```

The above command returns:

```
You are now connected to database "my_db" as user "pguser".
```

Create a new test table:

```PSQL
=# CREATE TABLE my_table (col1 integer, col2 varchar(50));
```

Insert a test row:

```PSQL
=# INSERT INTO my_table VALUES (1, 'foobar');
```

You have prepared a test database to back up. In practice, replace `my_db` with the name of your own database. Press `CTRL+D` and exit the `psql` command line.

## Use the `pg_dump` Utility to Create a Database Dump

Create a plain text dump of the database `my_db`:

```CONSOLE
$ pg_dump --no-owner --no-privileges -f /tmp/pgtemp/my_db_backup.sql my_db
```

This creates a file `my_db_backup.sql` in the directory `/tmp/pgtemp/` that you created earlier. Using the `--no-owner` and `--no-privileges` options allows any PostgreSQL username to restore the database on a fresh server. When these options are not specified, only the username that created the database can restore it. 

Verify that the dump is created:

```CONSOLE
$ ls -l /tmp/pgtemp/
```

The output should include the file `my_db_backup.sql`.

## Upload the Database Dump to Vultr Object Storage

Create a new bucket in Vultr Object Storage. This bucket is used to store the backup of the database dump. The command below creates a new bucket named `pgbackup`:

```CONSOLE
$ s3cmd mb s3://pgbackup
```

The output of the above command looks like this:

```
Bucket 's3://pgbackup/' created
```

Use `s3cmd` and upload the database dump to Object Storage:

```CONSOLE
$ s3cmd put /tmp/pgtemp/my_db_backup.sql s3://pgbackup/
```

The output of the above command looks like this:

```
upload: '/tmp/pgtemp/my_db_backup.sql' -> 's3://pgbackup/my_db_backup.sql'  [1 of 1]
913 of 913   100% in    0s    15.60 KB/s  done
```

This confirms that the database dump is saved to Vultr Object Storage. To verify that the dump is saved, check the contents of the remote directory: 

```CONSOLE
$ s3cmd ls s3://pgbackup
```

The output looks like the example below:

```
2024-04-10 16:32          913  s3://pgbackup/my_db_backup.sql
```

Notice that the output contains the dump file. Because the database is backed up, the local copy of the database dump is no longer needed. Delete it:

```CONSOLE
$ rm /tmp/pgtemp/my_db_backup.sql
```

By manually entering the commands, you have verified that all the steps of the process work as expected and can hence be automated.

## Automate the Backup Process with cron

To automate the process, include the above operations in a script and create a cron job to periodically run the script.

### Create a Bash Script

Write a Bash shell script that creates the backups as described above. The script consists of the following steps:

 * Declare variables to store the date, path of the dump file, name of the database, and path of the remote backup location.
 * Create a dump of the database using the `pg_dump` command.
 * Back up the dump to Vultr Object Storage.
 * Delete the local copy of the dump file.

Create a new Bash script in the home directory of the current user:

```CONSOLE
$ touch /home/pguser/pgbackup.sh
```

On Ubuntu and Debian based systems, for the example username `pguser`, the path of this Bash script is `/home/pguser/pgbackup.sh`. Modify this path based on your username and Operating System. 

Edit the Bash script using your preferred text editor, for example, Vim:

```CONSOLE
$ vim /home/pguser/pgbackup.sh
```

Add the following lines to the script:

```BASH
#!/usr/bin/bash  

DATE=$(date +"%Y-%m-%d")  

DUMP="/tmp/pgtemp/my_db_backup-$DATE.sql"

DATABASE=my_db  
BACKUP_LOCATION=pgbackup

pg_dump --no-owner --no-privileges -f $DUMP $DATABASE 
s3cmd put $DUMP s3://$BACKUP_LOCATION/  

rm $DUMP
```

Save and exit the editor.

The above script uses the following variables:

 * The `DATE` variable contains the current date, in the format `YYYY-MM-DD`, when the script is executed.
 * The `DUMP` variable denotes the filename of the database dump. The current date is appended to the end of the filename. Edit this variable to change the path and filename of the database dump. This example Bash script creates dump files that are named, for example, as `my_db-2024-03-31.sql`. 
 * The variable `DATABASE` stores the name of the database to back up. Change this variable to reflect the correct name of your database.
 * The variable `BACKUP_LOCATION` denotes the name of the Vultr Object Storage bucket where the dump files are backed up. In this example, dump files are backed up in the remote directory `s3://pgbackup/`. Edit this variable to back up at a different location. 

### Run and Test the Script

Make the script file executable:

```CONSOLE
$ chmod u+x /home/pguser/pgbackup.sh
```

Run the script:

```CONSOLE
$ /home/pguser/pgbackup.sh
```

The output of the script looks like the example shown below:

```
upload: '/tmp/pgtemp/my_db_backup-2024-04-10.sql' -> 's3://pgbackup/my_db_backup-2024-04-10.sql'  [1 of 1]
913 of 913   100% in    0s    24.29 KB/s  done
```

This shows that the dump is backed up to Vultr Object Storage. Check the Vultr Object Storage directory to verify that the new backup is uploaded:

``` CONSOLE
$ s3cmd ls s3://pgbackup
```

The output should contain the backed up dump file. After checking that the script works, automate its execution using cron.

### Add a New cron Job to Run the Script

To add a cron job, edit the cron configuration for the current user:

``` CONSOLE
$ crontab -e
```

Add this line at the end of the file:

```
0 2 * * * /home/pguser/pgbackup.sh                       
```

Save and exit the file. Check that the new job is added:

```CONSOLE
$ crontab -l
```

In the cron job specification, as shown above, the first five fields are:

 * Minute, numbered from 0 to 59
 * Hour, numbered from 0 to 23
 * Day of Month, numbered from 1 to 31
 * Month Number, numbered from 1 to 12
 * Day of Week, numbered from 0 to 7 

These fields decide the day, the date, and the time when the job runs. For example, specifying a value of `1` for the Day of Month means that job runs on the first day of every month. Using an asterisk in any field means the job is run for all values of the field. For example, using an asterisk for the Month Number means the job is run in all the months from January to December. 

The last field in the specification is: 

 * The path to the command or file to execute

Thus, the example cron job runs the backup script every day of every week of every month at 2:00 AM. Consider running backup and maintenance tasks when the server is expected not to be busy with user requests. 

## Conclusion

In this guide, you executed the following steps:

 * Create a plain text dump of a PostgreSQL database. 
 * Manually back up the dump to Vultr Object Storage.
 * Write a Bash script consisting of the above two steps.
 * Automate the execution of the script using a cron job. 

To implement this system in practice, extend it to run the script at daily, weekly, and monthly intervals. The [guide about using the cron task scheduling system](https://docs.vultr.com/how-to-use-the-cron-task-scheduler) explains how to do that. Also modify the filenames and storage paths of the backups to reflect whether it is a daily, weekly, or monthly backup. For example, use different buckets for weekly and monthly backups.

### Learn More

 * This guide shows the basic command to create a dump. [Learn how to use `pg_dump` to backup and restore PostgreSQL databases](https://docs.vultr.com/how-to-backup-and-restore-postgresql-databases-with-pg-dump). 
 * This guide includes the necessary `s3cmd` command to push the backup to Object storage. [Know how to use the `s3cmd` command line application with Vultr Object Storage](https://docs.vultr.com/how-to-use-s3cmd-with-vultr-object-storage). 

