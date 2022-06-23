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

###1. Add inboud rule to Amazon RDS MariaDB instance Security Group

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

###2. Find Amazon RDS MariaDB connection info

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
        "(\"Endpoint : \" + .DBInstances[0].Endpoint.Address), (\"Port : \" + (.DBInstances[0].Endpoint.Port | tostring)), (\"Master Username : \" + .DBInstances[0].MasterUsername), (\"DB Name : \" + .DBInstances[0].DBName)"
    ```
    
3. Put DB instance endpoint address to a variable

    ```console
    DB_INSTANCE_ENDPOINT=$(aws rds describe-db-instances \
      --db-instance-identifier targetdb-maria \
      --query "DBInstances[0].Endpoint.Address" \
      --output text)
    echo $DB_INSTANCE_ENDPOINT
    ```

###3. Connect to Amazon RDS MariaDB instance using MariaDB Client 

1. Save Password to .pgpass file
    ```console
    (umask 177 ; \
      echo $DB_INSTANCE_ENDPOINT:45432:postgres:postgres:target1234 > .pgpass)
    cat .pgpass
    ```

2. Connect to instance
    ```console
    psql \
      --host=$DB_INSTANCE_ENDPOINT \
      --port=45432 \
      --username=postgres \
      --no-password \
      --dbname=postgres
    ```
    
3. MariaDB Client provides a prompt with the name of the database
    ```console
    psql (13.3, server 13.4)
    SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
    Type "help" for help.

    postgres=>
    ```


## 5. Create a user, database, schema on Amazon RDS MariaDB

###1. Create a user with a password

    ```sql
    CREATE USER addrdba
    WITH PASSWORD '1234';
    ```
    
    List User
    ```sql
    SELECT usename, usecreatedb FROM pg_catalog.pg_user;
    ```
    
    List Roles
    ```sql
    \du
    ```

###2. Create a target database

1. Create a database

    ```sql
    CREATE DATABASE addr_target
    TEMPLATE = template0
    ENCODING = 'UTF8'
    LC_COLLATE = 'C'
    LC_CTYPE = 'C';
    ```
    
    Change database owner to created user
    ```sql
    ALTER DATABASE addr_target
    OWNER TO addrdba;
    ```
    
    List Databases
    ```sql
    \list
    ```

3. Create a schema on a database 

    Connect as a user created
    ```sql
    \connect addr_target addrdba
    ```
    User Password
    ```text
    1234
    ```
    
    ```console
    postgres=> \connect addr_target addrdba
    Password for user addrdba: 
    psql (13.3, server 13.4)
    SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
    You are now connected to database "addr_target" as user "addrdba".
    addr_target=>
    ```
    
    Create a schema on a newly created database 
    ```sql
    CREATE SCHEMA addr
    AUTHORIZATION addrdba;
    ```
    
    List schemas
    ```sql
    \dn
    ```
    
    ```sql
    addr_target=> \dn
      List of schemas
      Name  |  Owner   
    --------+----------
     addr   | addrdba
     public | rdsadmin
    (2 rows)
    ```
