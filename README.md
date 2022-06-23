# Amazon RDS PostgreSQL with AWS CloudFormation


## 1. Launch AWS CloudShell

Refer to [AWS CloudShell](https://github.com/t2yijaeho/AWS-CloudShell)


## 2. Create an Amazon RDS PostgreSQL

1. Get an AWS CloudFormation stack template body

    ```bash
    wget https://github.com/t2yijaeho/Amazon-RDS-PostgreSQL-with-AWS-CloudFormation/raw/matia/Template/RDS-PostgreSQL.yaml
    ```


2. Create an AWS CloudFormation stack

    ```bash
    aws cloudformation create-stack \
      --stack-name RDS-PostgreSQL \
      --template-body file://./RDS-PostgreSQL.yaml
    ```

3. AWS CloudFormation returns following output

    ```json
    {
    "StackId": "arn:aws:cloudformation:us-abcd-x:123456789012:stack/RDS-PostgreSQL/b4d0f5e0-d4c2-11ec-9529-06edcc65f112"
    }
    ```

4. Monitor the progress by the stack's events in AWS management console

    <img src="https://github.com/t2yijaeho/Amazon-RDS-PostgreSQL-with-AWS-CloudFormation/blob/matia/images/CloudFormation%20Stack%20Creation%20Events.png?raw=true">


## 3. Install PostgreSQL Client

1. List the available PostgreSQL toics from the Extras Library

    ```bash
    sudo amazon-linux-extras | grep postgresql
    ```

2. Enable the desired latest PostgreSQL topic

    ```bash
    sudo amazon-linux-extras enable postgresql13 | grep postgresql
    ```

2. Install PostgreSQL topic

    ```bash
    sudo yum clean metadata && sudo yum install -y postgresql
    ```

3. Verify the installation and confirm the PostgreSQL Client version:

    ```bash
    sudo yum list installed postgresql
    ```
    ```bash
    psql --version
    ```


## 4. Connect to an Amazon RDS PostgreSQL

1. Add inboud rule to Amazon RDS PostgreSQL Security Group

    Find CloudShell IP address
    ```bash
    CLOUDSHELL_IP_ADDRESS=$(curl https://checkip.amazonaws.com)
    echo $CLOUDSHELL_IP_ADDRESS
    ```
    
    Find Amazon RDS PostgreSQL Security Group ID
    ```bash
    RDS_SECURITY_GROUP_ID=$(aws rds describe-db-instances \
        --db-instance-identifier targetdb \
      | jq --raw-output \
        ".DBInstances[0].VpcSecurityGroups[0].VpcSecurityGroupId")
    echo $RDS_SECURITY_GROUP_ID
    ```
    
    Add CloudShell IP address to Security Group inboud rule
    ```bash
    aws ec2 authorize-security-group-ingress \
      --group-id $RDS_SECURITY_GROUP_ID \
      --protocol tcp --port 45432 \
      --cidr "${CLOUDSHELL_IP_ADDRESS}/32"
    ```

2. Find Amazon RDS PostgreSQL connection info

    Describe Amazon RDS PostgreSQL instance 
    ```bash
    aws rds describe-db-instances \
        --db-instance-identifier targetdb
    ```
    
    Find Endpoint, Port, Username, Database Name
    ```bash
    aws rds describe-db-instances \
        --db-instance-identifier targetdb \
      | jq --raw-output \
        "(\"Endpoint : \" + .DBInstances[0].Endpoint.Address), (\"Port : \" + (.DBInstances[0].Endpoint.Port | tostring)), (\"Master Username : \" + .DBInstances[0].MasterUsername), (\"DB Name : \" + .DBInstances[0].DBName)"
    ```
    
    Put DB instance endpoint address to a variable
    ```bash
    DB_INSTANCE_ENDPOINT=$( \
      aws rds describe-db-instances \
        --db-instance-identifier targetdb \
      | jq --raw-output \
        '.DBInstances[0].Endpoint.Address')
    echo $DB_INSTANCE_ENDPOINT
    ```

3. Connect to Amazon RDS PostgreSQL instance using PostgreSQL Client 

    Save Password to .pgpass file
    ```bash
    (umask 177 ; \
      echo $DB_INSTANCE_ENDPOINT:45432:postgres:postgres:target1234 > .pgpass)
    cat .pgpass
    ```

    Connect to instance
    ```bash
    psql \
      --host=$DB_INSTANCE_ENDPOINT \
      --port=45432 \
      --username=postgres \
      --no-password \
      --dbname=postgres
    ```
    
    PostgreSQL Client provides a prompt with the name of the database
    ```bash
    psql (13.3, server 13.4)
    SSL connection (protocol: TLSv1.2, cipher: ECDHE-RSA-AES256-GCM-SHA384, bits: 256, compression: off)
    Type "help" for help.

    postgres=>
    ```


## 5. Create a user, database, schema on Amazon RDS PostgreSQL

1. Create a user with a password

    ```sql
    CREATE USER addrdba
    WITH PASSWORD '1234';
    ```
    
    List User
    ```sql
    SELECT usename, usecreatedb FROM pg_catalog.pg_user;
    ```
    <img src="https://github.com/t2yijaeho/Amazon-RDS-PostgreSQL-with-AWS-CloudFormation/blob/matia/images/PostgreSQL%20-%20List%20of%20pg_user.png?raw=true">    
    
    List Roles
    ```sql
    \du
    ```
    <img src="https://github.com/t2yijaeho/Amazon-RDS-PostgreSQL-with-AWS-CloudFormation/blob/matia/images/PostgreSQL%20-%20List%20of%20roles.png?raw=true">

2. Create a target database

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
    <img src="https://github.com/t2yijaeho/Amazon-RDS-PostgreSQL-with-AWS-CloudFormation/blob/matia/images/PostgreSQL%20-%20List%20of%20Databases.png?raw=true">

3. Create a schema on a database 

    Connect as a user created
    ```sql
    \connect addr_target addrdba
    ```
    User Password
    ```text
    1234
    ```
    
    ```bash
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
