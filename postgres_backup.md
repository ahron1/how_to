# How to Automatically Back Up PostgreSQL Databases with Object Storage

## Introduction

Backing up the database to the cloud allows you to restore your data in case of contingencies. Vultr Object Storage is a cloud object storage system. Using Vultr Object Storage for backups gives you the flexibility to dynamically deploy more storage as your needs increase. s3cmd is a command-line application used to manage Vultr Object Storage. To create backups of PostgreSQL databases, you can use tools like `pg_dump` and `pg_dumpall`. These tools generate a dump file consisting of SQL commands, either in plain text format or in archived format. `pg_dump` backs up a single database whereas `pg_dumpall` backs up all the databases in the cluster.

This article explains how you can 
 * Backup a PostgreSQL database using `pg_dump` 
 * Upload the backup to Vultr Object Storage
 * Automate the process of backing up PostgreSQL databases to Vultr Object Storage

## Prerequisites

To follow the instructions in this guide, you may need to:

 * [Install PostgreSQL](https://docs.vultr.com/how-to-install-postgresql-database-server-on-ubuntu-22-04)
 * [Create an instance of Vultr Object Storage](https://docs.vultr.com/vultr-object-storage)
 * [Configure s3cmd to use it with Vultr Object Storage](https://docs.vultr.com/how-to-use-s3cmd-with-vultr-object-storage#set-default-configuration)

### Create New PostgreSQL User and Test Database

Create a new PostgreSQL user. Then, create an example database to test the backup process described in this article.

 1. Log in to the PSQL command line as the PostgreSQL root user.
    ```CONSOLE
    $ sudo -u postgres psql
    ```
 1. Create a new PostgreSQL user, `pguser`. Replace `pguser` with your preferred username for logging in to PostgreSQL and `your-password` with a strong password.
    ```PSQL
    psql > CREATE ROLE pguser WITH LOGIN PASSWORD 'your-password' ;
    ```
 1. Create a test database, `my_db`. Replace `my_db` with the desired name of your database.
    ```PSQL
    =# CREATE DATABASE my_db;
    ```
 1. Grant full privileges on the new database to the new user.
    ```PSQL
    psql > GRANT ALL PRIVILEGES ON DATABASE my_db TO pguser;
    ```
 1. Exit the PSQL console.
    ```PSQL
    psql > exit
    ```
 1. Create a `.pgpass` file in the home directory of the user that will run `pg_dump`. The `.pgpass` file stores the login credentials. Using the `.pgpass` file, you can run commands from shell scripts without manually entering passwords.
    ```CONSOLE
    $ cd
    $ touch .pgpass
    ```
 1. Open the `.pgpass` file using a text editor, such as Nano.
    ```CONSOLE
    $ nano .pgpass
    ```
 1. Credentials in the `.pgpass` file use the following syntax:
    ```
    hostname:port:database:username:password
    ```
    Add a line with the user's credentials to the `.pgpass` file. Replace these values with the actual database details.
    ```
    localhost:5432:my_db:pguser:your-password
    ```
 1. Save the file and exit the editor.
 1. Set the file's permissions to `0600`.
    ```CONSOLE
    $ chmod 600 .pgpass
    ```
 1. Log in to PSQL as the new user and connect to the new database. 
    ```CONSOLE
    $ psql --host localhost --port 5432 --username pguser --dbname my_db 
    ```
 1. Create a new test table.
    ```PSQL
    =# CREATE TABLE my_table (col1 integer, col2 varchar(50));
    ```
 1. Insert a test row.
    ```PSQL
    =# INSERT INTO my_table VALUES (1, 'foobar');
    ```
 1. Exit the PSQL console.
    ```PSQL
    psql > exit
    ```

In this section, you created a new user and prepared a test database to back up. 

## Use the `pg_dump` Utility to Create a Database Dump

Create a plain text dump of the database `my_db`. Replace `my_db` with your actual database name and `backup.dump` with the desired filename of the backup.

```CONSOLE
$ pg_dump --host=localhost --port=5432 --username=pguser --dbname=my_db --file=backup.dump 
```

## Upload the Database Dump to Vultr Object Storage

 1. Create a new bucket in Vultr Object Storage to store the backup of the database dump. Replace `pgbackup` with the desired name of the bucket.
    ```CONSOLE
    $ s3cmd mb s3://pgbackup
    ```
    Output: 
    ```
    Bucket 's3://pgbackup/' created
    ```
 1. Use `s3cmd` and upload the database dump to Object Storage. 
    ```CONSOLE
    $ s3cmd put backup.dump s3://pgbackup/
    ```

    Output: 
    ```
    upload: 'backup.dump' -> 's3://pgbackup/backup.dump'  [1 of 1]
    913 of 913   100% in    0s    15.60 KB/s  done
    ```
 1. Check the contents of the remote directory. 
    ```CONSOLE
    $ s3cmd ls s3://pgbackup
    ```
    Output: 
    ```
    2024-04-10 16:32          913  s3://pgbackup/backup.dump
    ```
 1. After backing up the database dump to Vultr Object Storage, the local copy of the database dump is redundant. Delete it to save disk space.
    ```CONSOLE
    $ rm backup.dump
    ```

In this section, you manually created a dump of the database and uploaded it to Vultr Object Storage.

## Automate the Backup Process with cron

To automate the backup process, include the operations of the previous sections in a script and create a cron job to periodically run the script.

### Create a Bash Script

 1. Create a new Bash script. Replace `script.sh` with the actual filename of the script.
    ```CONSOLE
    $ touch script.sh
    ```
 1. Edit the Bash script using your preferred text editor, for example, Nano.
    ```CONSOLE
    $ nano script.sh
    ```
 1. Add the following lines to the script. Replace the variables with their actual values.
    ```BASH
    #!/usr/bin/bash  

    HOST=localhost
    PORT=5432 
    USER=pguser 
    DATABASE=my_db  
    DUMP="backup-$(date +"%Y-%m-%d-%H-%M").dump"
    BACKUP_LOCATION=pgbackup

    pg_dump --host=$HOST --port=$PORT --username=$USER --dbname=$DATABASE --file=$DUMP 
    s3cmd put $DUMP s3://$BACKUP_LOCATION/  

    rm $DUMP
    ```
    In this script, modify the following variables to reflect your desired values:

     * `HOST` is the hostname.
     * `PORT` is the port number.
     * `USER` is the username.
     * `DATABASE` represents the name of the database to back up. 
     * `DUMP` denotes the name of the dump file in the format `backup-year-month-day-hour-minute.dump`. This format helps to identify the date and time when the backup was created.      
     * `BACKUP_LOCATION` is the name of the Vultr Object Storage bucket where the dump files are backed up. 

 1. Save and exit the editor.

 1. Make the script file executable.
    ```CONSOLE
    $ chmod u+x script.sh
    ```
 1. Run the script.
    ```CONSOLE
    $ bash script.sh
    ```
    Output:
    ```
    upload: 'backup-2024-04-10-13-22-dump' -> 's3://pgbackup/backup-2024-04-10-13-22-dump'  [1 of 1]
    913 of 913   100% in    0s    24.29 KB/s  done
    ```
 1. Check the contents of the remote directory to verify that the script works.
    ```CONSOLE
    $ s3cmd ls s3://pgbackup
    ```
    Output: 
    ```
    2024-04-10 13:22          913  s3://pgbackup/backup-2024-04-10-13-22.dump

### Add a New cron Job to Run the Script

Use cron to automatically execute the backup script. 

 1. To add a cron job, edit the cron configuration for the current user.
    ``` CONSOLE
    $ crontab -e
    ```
 1. cron entries use the following syntax:
    ```
    Minute     Hour   Day     Month     Day-of-week   command
    ```
    Add a cron task to run the backup script every day of every week of every month at 2:00 AM. Replace these values with the actual database details.
    ```
    0 2 * * * /path/to/script.sh                       
    ```

    Modify the above line to reflect your desired frequency of backups. The following fields decide the day, the date, and the time when the job runs:

     * Minute, numbered from 0 to 59
     * Hour, numbered from 0 to 23
     * Day, numbered from 1 to 31
     * Month, numbered from 1 to 12
     * Day of Week, numbered from 0 to 7 

     To run the job for all values of a field, use an asterisk for the field value. For example, to run a job in all the months, use an asterisk for the Month Number.

    The last field in the cron job syntax is: 

     * The path to the command or file to execute. Use the filename with the complete path.

 1. Save and close the editor. 

 1. Check that the new job is added.
    ```CONSOLE
    $ crontab -l
    ```
    Output:
    ```
    0 2 * * * /path/to/script.sh
    ```
## Conclusion

In this guide, you executed the following steps:

 * Create a plain text dump of a PostgreSQL database. 
 * Manually back up the dump to Vultr Object Storage.
 * Write a Bash script consisting of the above two steps.
 * Automate the execution of the script using a cron job. 

