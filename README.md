# Amazon RDS MariaDB with AWS CloudFormation


## 1. Launch AWS CloudShell

Refer to [AWS CloudShell](https://github.com/t2yijaeho/AWS-CloudShell)


## 2. Create an Amazon RDS MariaDB


1. Get an AWS CloudFormation stack template body


    ```script
    wget https://github.com/t2yijaeho/Amazon-RDS-MariaDB-with-CloudFormation/raw/matia/Template/RDS-MariaDB.yaml
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ wget https://github.com/t2yijaeho/Amazon-RDS-MariaDB-with-CloudFormation/raw/matia/Template/RDS-MariaDB.yaml
    ...
    100%[=============================================================================================>] 1,582       --.-K/s   in 0s      

    2022-06-23 06:03:51 (28.8 MB/s) - ‘RDS-MariaDB.yaml’ saved [1582/1582]
    
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


2. Create an AWS CloudFormation stack


    ```console
    aws cloudformation create-stack \
      --stack-name RDS-MariaDB \
      --template-body file://./RDS-MariaDB.yaml
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ aws cloudformation create-stack \
    >   --stack-name RDS-MariaDB \
    >   --template-body file://./RDS-MariaDB.yaml
    {
    "StackId": "arn:aws:cloudformation:us-abcd-x:123456789012:stack/RDS-MariaDB/a1b2c3d4-e5f6-78gh-9012-34ijkl56m789"
    }
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


3. Monitor the progress by the stack's events in AWS management console


    <img src="https://github.com/t2yijaeho/Amazon-RDS-MariaDB-with-CloudFormation/blob/matia/images/CloudFormation%20Stack%20Creation%20Events.png?raw=true">


## 3. Install MariaDB Client


1. List the available MariaDB toics from the Extras Library


    ```console
    sudo amazon-linux-extras | grep mariadb
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ sudo amazon-linux-extras | grep mariadb
     54  mariadb10.5              available    [ =stable ]
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


2. Enable the desired latest MariaDB topic


    ```console
    sudo amazon-linux-extras enable mariadb10.5 | grep mariadb
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ sudo amazon-linux-extras enable mariadb10.5 | grep mariadb
     54  mariadb10.5=latest       enabled      [ =stable ]
     # yum install mariadb
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


2. Install MariaDB topic


    ```console
    sudo yum clean metadata && sudo yum install -y mariadb
    ```

    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ sudo yum clean metadata && sudo yum install -y mariadb
    ...
    Updated:
      mariadb.x86_64 3:10.5.10-2.amzn2.0.1                                                                                                 

    Dependency Updated:
      mariadb-libs.x86_64 3:10.5.10-2.amzn2.0.1                                                                                            

    Complete!
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


3. Verify the installation and confirm the MariaDB Client version:


    ```console
    sudo yum list installed mariadb
    mariadb --version
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ sudo yum list installed mariadb
     Loaded plugins: ovl, priorities
    Installed Packages
    mariadb.x86_64                                      3:10.5.10-2.amzn2.0.1                                       @amzn2extra-mariadb10.5
    [cloudshell-user@ip-10-0-123-234 ~]$ mariadb --version
    mariadb  Ver 15.1 Distrib 10.5.10-MariaDB, for Linux (x86_64) using  EditLine wrapper
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


## 4. Connect to an Amazon RDS MariaDB


### 1. Add inboud rule to Amazon RDS MariaDB instance Security Group


1. Find CloudShell IP address


    ```console
    CLOUDSHELL_IP_ADDRESS=$(curl https://checkip.amazonaws.com)
    echo $CLOUDSHELL_IP_ADDRESS
    ``` 

    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ CLOUDSHELL_IP_ADDRESS=$(curl https://checkip.amazonaws.com)
      % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
    100    14  100    14    0     0     49      0 --:--:-- --:--:-- --:--:--    49
    [cloudshell-user@ip-10-0-123-234 ~]$ echo $CLOUDSHELL_IP_ADDRESS
    56.123.234.78
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ``` 


2. Find Amazon RDS MariaDB Security Group ID


    ```console
    RDS_SECURITY_GROUP_ID=$(aws rds describe-db-instances \
      --db-instance-identifier targetdb-maria \
      --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" \
      --output text)
    echo $RDS_SECURITY_GROUP_ID
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ RDS_SECURITY_GROUP_ID=$(aws rds describe-db-instances \
    >       --db-instance-identifier targetdb-maria \
    >       --query "DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId" \
    >       --output text)
    [cloudshell-user@ip-10-0-123-234 ~]$ echo $RDS_SECURITY_GROUP_ID
    sg-01a234b567cd890ef
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ``` 


3. Add CloudShell IP address to Security Group inboud rule


    ```console
    aws ec2 authorize-security-group-ingress \
      --group-id $RDS_SECURITY_GROUP_ID \
      --protocol tcp --port 33306 \
      --cidr "${CLOUDSHELL_IP_ADDRESS}/32"
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ aws ec2 authorize-security-group-ingress \
    >   --group-id $RDS_SECURITY_GROUP_ID \
    >   --protocol tcp --port 33306 \
    >   --cidr "${CLOUDSHELL_IP_ADDRESS}/32"
    {
        "Return": true,
        "SecurityGroupRules": [
            {
                "SecurityGroupRuleId": "sgr-1a2b34c56def7g890",
                "GroupId": "sg-01a234b567cd890ef",
                "GroupOwnerId": "123456789012",
                "IsEgress": false,
                "IpProtocol": "tcp",
                "FromPort": 33306,
                "ToPort": 33306,
                "CidrIpv4": "123.234.234.33/32"
            }
        ]
    }
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ``` 


### 2. Find Amazon RDS MariaDB connection info


1. Describe Amazon RDS MariaDB instance 


    ```console
    aws rds describe-db-instances \
        --db-instance-identifier targetdb-maria
    ```


2. Find Endpoint, Port, Username, Database Name


    ```console
    aws rds describe-db-instances \
        --db-instance-identifier targetdb-maria \
      | jq --raw-output \
        "(\"Endpoint : \" + .DBInstances[0].Endpoint.Address), \
         (\"Port : \" + (.DBInstances[0].Endpoint.Port | tostring)), \
         (\"Master Username : \" + .DBInstances[0].MasterUsername), \
         (\"DB Name : \" + .DBInstances[0].DBName)"
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ aws rds describe-db-instances \
    >     --db-instance-identifier targetdb-maria \
    >   | jq --raw-output \
    >     "(\"Endpoint : \" + .DBInstances[0].Endpoint.Address), \
    >      (\"Port : \" + (.DBInstances[0].Endpoint.Port | tostring)), \
    >      (\"Master Username : \" + .DBInstances[0].MasterUsername), \
    >      (\"DB Name : \" + .DBInstances[0].DBName)"
    Endpoint : targetdb-maria.ab3cdefghijk.us-west-x.rds.amazonaws.com
    Port : 33306
    Master Username : admin
    DB Name : 
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ``` 


3. Put DB instance endpoint address to a variable


    ```console
    DB_INSTANCE_ENDPOINT=$(aws rds describe-db-instances \
      --db-instance-identifier targetdb-maria \
      --query "DBInstances[0].Endpoint.Address" \
      --output text)
    echo $DB_INSTANCE_ENDPOINT
    ```
    
    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ DB_INSTANCE_ENDPOINT=$(aws rds describe-db-instances \
    >   --db-instance-identifier targetdb-maria \
    >   --query "DBInstances[0].Endpoint.Address" \
    >   --output text)
    [cloudshell-user@ip-10-0-123-234 ~]$ echo $DB_INSTANCE_ENDPOINT
    targetdb-maria.ab3cdefghijk.us-west-x.rds.amazonaws.com
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ``` 


### 3. Connect to Amazon RDS MariaDB instance using MariaDB Client 


1. Connect to instance then MariaDB Client provides a prompt

    ```console
    mariadb -h $DB_INSTANCE_ENDPOINT  -P 33306 -u admin  --password=target1234
    ```
    
   ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ mariadb -h $DB_INSTANCE_ENDPOINT  -P 33306 -u admin  --password=target1234
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 64
    Server version: 10.6.7-MariaDB managed by https://aws.amazon.com/rds/

    Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> 
    ``` 


## 5. Create a user, database, schema on Amazon RDS MariaDB


### 1. Create a target database


1. Create a database


    ```sql
    CREATE DATABASE ADDR
    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    ```

    ```sql
    MariaDB [(none)]> CREATE DATABASE ADDR
    ->     CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    Query OK, 1 row affected (0.002 sec)

    MariaDB [(none)]> 
    ```

2. Set the current default database


    ```sql
    USE ADDR;
    ```

    ```sql
    MariaDB [(none)]> USE ADDR;
    Database changed
    MariaDB [ADDR]>  
    ```


### 2. Create a user


1. Create a user with a password


    ```sql
    CREATE USER ADDRDBA
    IDENTIFIED BY '1234';
    ```
    
    ```sql
    MariaDB [ADDR]> CREATE USER ADDRDBA
    -> IDENTIFIED BY '1234';
    Query OK, 0 rows affected (0.003 sec)

    MariaDB [ADDR]> 
    ```
    
    
2. List User


    ```sql
    SELECT User FROM mysql.user;
    ```

    ```sql
    MariaDB [ADDR]> SELECT User FROM mysql.user;
    +-------------+
    | User        |
    +-------------+
    | ADDRDBA     |
    | admin       |
    | mariadb.sys |
    | rdsadmin    |
    +-------------+
    4 rows in set (0.002 sec)

    MariaDB [ADDR]> 
    ```


3. Add privileges to user


    ```sql
    GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES 
    ON ADDR.* 
    TO ADDRDBA;
    ```
    
    ```sql
    MariaDB [ADDR]> GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES 
    -> ON ADDR.* 
    -> TO ADDRDBA;
    Query OK, 0 rows affected (0.001 sec)

    MariaDB [ADDR]> 
    ```
    

4. List privileges


    ```sql
    show grants for 'ADDRDBA';
    ```

    ```sql
    MariaDB [ADDR]> show grants for 'ADDRDBA';
    +-------------------------------------------------------------------------------------------------------------------------------------+
    | Grants for ADDRDBA@%                                                                                                                |
    +-------------------------------------------------------------------------------------------------------------------------------------+
    | GRANT USAGE ON *.* TO `ADDRDBA`@`%` IDENTIFIED BY PASSWORD '*A4B6157319038724E3560894F7F932C8886EBFCF'                              |
    | GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES ON `ADDR`.* TO `ADDRDBA`@`%` |
    +-------------------------------------------------------------------------------------------------------------------------------------+
    2 rows in set (0.001 sec)

    MariaDB [ADDR]> 
    ```


### 3. Create a schema on a database 


1. Exit to CloudShell


    ```sql
    exit
    ```
    
    ```sql
    MariaDB [ADDR]> exit
    Bye
    [cloudshell-user@ip-10-0-123-234 ~]$ 
    ```


2. Connect as the user created


    ```console
    mariadb -h $DB_INSTANCE_ENDPOINT -P 33306 -u ADDRDBA --password=1234
    ```

    ```console
    [cloudshell-user@ip-10-0-123-234 ~]$ mariadb -h $DB_INSTANCE_ENDPOINT -P 33306 -u ADDRDBA --password=1234
    Welcome to the MariaDB monitor.  Commands end with ; or \g.
    Your MariaDB connection id is 98
    Server version: 10.6.7-MariaDB managed by https://aws.amazon.com/rds/

    Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

    Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

    MariaDB [(none)]> 
    
    ```
    

3. Verify case-sensitive manner


    ```sql
    SHOW GLOBAL VARIABLES LIKE 'lower_case_table_names';
    ```
    
    ```sql
    MariaDB [(none)]> SHOW GLOBAL VARIABLES LIKE 'lower_case_table_names';
    +------------------------+-------+
    | Variable_name          | Value |
    +------------------------+-------+
    | lower_case_table_names | 0     |
    +------------------------+-------+
    1 row in set (0.002 sec)

    MariaDB [(none)]> 
    ```
    
