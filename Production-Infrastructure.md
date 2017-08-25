## Overview
Every component (apart from the database) in this stack is packaged as a docker container. Because of this, it can be deployed to any infrastructure capable of running containers. In this section we'll explain how to setup [Amazon EC2 Container Service (ECS)](https://aws.amazon.com/ecs/) to run your containers and setup a PostgreSQL database is [Amazon Relational Database Service (RDS)](https://aws.amazon.com/rds/) that will hold your data (and most of the code).

## Familiarize yourself with ECS concepts 
https://aws.amazon.com/ecs/
https://aws.amazon.com/ecs/getting-started/
http://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html


## ECS Cluster
Create your cluster using the wizard (start with one ec2 instance)
In this example the name used is `mycluster`
http://docs.aws.amazon.com/AmazonECS/latest/developerguide/create_cluster.html

```sh
export CLUSTER_NAME=mycluster
```

Get the cluster's cloudformation stack name:
```sh
aws cloudformation list-stacks --output table --query 'StackSummaries[*].[StackName,TemplateDescription]'
```

Result should look something like this
```
--------------------------------------------------------------------------------------------------------------------------------------------------------
|                                                                      ListStacks                                                                      |
+-------------------------------+----------------------------------------------------------------------------------------------------------------------+
|  EC2ContainerService-mycluster|  AWS CloudFormation template to create a new VPC or use an existing VPC for ECS deployment in Create Cluster Wizard  |
+-------------------------------+----------------------------------------------------------------------------------------------------------------------+
```

Save the stack name to env
```sh
export STACK_NAME=EC2ContainerService-mycluster
```

Extract stack configuration info
```sh
while read k v ; do export Cluster_Resource_$k=$v; done < <( \
    aws cloudformation describe-stack-resources \
        --stack-name $STACK_NAME \
        --output text \
        --query 'StackResources[*].[LogicalResourceId, PhysicalResourceId]'\
)

while read k v ; do export Cluster_Param_$k=$v; done < <( \
    aws cloudformation describe-stacks \
        --stack-name $STACK_NAME \
        --output text \
        --query 'Stacks[*].Parameters[*][ParameterKey,ParameterValue]'\
)

export AWS_REGION=`echo $Cluster_Param_VpcAvailabilityZones | cut -d',' -f1 | head --bytes -2`

#check if it worked with
env | grep Cluster
echo $AWS_REGION

```



## SSL Certificate
Create a certificate (or upload) in AWS Certificate Manager
http://docs.aws.amazon.com/acm/latest/userguide/acm-overview.html

```sh
# List the certificates
aws acm list-certificates

# Save the ARN
export CERTIFICATE_ARN="arn:aws:acm:us-east-1:CHANGE-WITH-YOURS:certificate/CHANGE-WITH-YOURS"
```

Result should look like this
```sh
------------------------------------------------------------------------------------------------------------
|                                             ListCertificates                                             |
+----------------------------------------------------------------------------------------------------------+
||                                         CertificateSummaryList                                         ||
|+---------------------------------------------------------------------------------------+----------------+|
||                                    CertificateArn                                     |  DomainName    ||
|+---------------------------------------------------------------------------------------+----------------+|
||  arn:aws:acm:us-east-1:000000000000:certificate/00000000-0000-0000-0000-000000000000  |  mydomain.com  ||
|+---------------------------------------------------------------------------------------+----------------+|
```

## Loadbalancer

The loadbalancer will route traffic to our containers
(just like the cluster, it can be used for multiple applications)

```sh
aws cloudformation create-stack \
    --stack-name $CLUSTER_NAME-loadbalancer \
    --template-body file://cloudformation/loadbalancer.yml \
    --capabilities CAPABILITY_IAM \
    --parameters \
    ParameterKey=ClusterName,ParameterValue=$CLUSTER_NAME \
    ParameterKey=CertificateArn,ParameterValue=$CERTIFICATE_ARN \
    ParameterKey=Vpc,ParameterValue=$Cluster_Resource_Vpc \
    ParameterKey=EcsSecurityGroup,ParameterValue=$Cluster_Resource_EcsSecurityGroup \
    ParameterKey=PubSubnetAz1,ParameterValue=$Cluster_Resource_PubSubnetAz1 \
    ParameterKey=PubSubnetAz2,ParameterValue=$Cluster_Resource_PubSubnetAz2 \
```

## Image Repository

We'll store our OpenResty image (the only one that changes) in [Amazon EC2 Container Registry](https://aws.amazon.com/ecr/).
You can use any docker image repository you like.

```sh
# read project env vars
source .env

# create the repository
aws ecr create-repository --repository-name $COMPOSE_PROJECT_NAME/openresty

# extract the uri in a separate env var
export OPENRESTY_REPO_URI=`aws ecr describe-repositories --repository-name $COMPOSE_PROJECT_NAME/openresty --output text --query 'repositories[0].repositoryUri'`
```


## Database (PostgreSQL in RDS)

```sh
# read project env vars
source .env

# set the subnet to the same VPS as the cluster
aws rds create-db-subnet-group \
    --db-subnet-group-name $COMPOSE_PROJECT_NAME-db-subnet \
    --db-subnet-group-description $COMPOSE_PROJECT_NAME-db-subnet \
    --subnet-ids $Cluster_Resource_PubSubnetAz1 $Cluster_Resource_PubSubnetAz2


# allow ECS nodes to connect to this db's in the same security group as the cluster
aws ec2 authorize-security-group-ingress \
              --region $AWS_REGION \
              --group-id $Cluster_Resource_EcsSecurityGroup \
              --protocol tcp \
              --port 5432 \
              --source-group $Cluster_Resource_EcsSecurityGroup

# allow yourself to connect directly to the database
export MY_IP=`dig +short myip.opendns.com @resolver1.opendns.com`
aws ec2 authorize-security-group-ingress \
              --region $AWS_REGION \
              --group-id $Cluster_Resource_EcsSecurityGroup \
              --protocol tcp \
              --port 5432 \
              --cidr $MY_IP/32

# create the database
aws rds create-db-instance \
    --db-instance-identifier $COMPOSE_PROJECT_NAME-db \
    --db-name $DB_NAME \
    --vpc-security-group-ids $Cluster_Resource_EcsSecurityGroup \
    --allocated-storage 20 \
    --db-instance-class db.t2.micro \
    --engine postgres \
    --publicly-accessible \
    --multi-az \
    --db-subnet-group-name $COMPOSE_PROJECT_NAME-db-subnet \
    --master-username $SUPER_USER \
    --master-user-password SET-YOUR-ROOT-PASSWORD-HERE

# check the AWS management pannel adn wait until the database is "ready"

# export production db host
export PRODUCTION_DB_HOST=`aws rds describe-db-instances --db-instance-identifier $COMPOSE_PROJECT_NAME-db --output text --query 'DBInstances[0].Endpoint.Address'`

# check you can connect
psql -h $PRODUCTION_DB_HOST -U $SUPER_USER $DB_NAME

# create the authenticator role used by PostgREST to connect
psql \
    -h $PRODUCTION_DB_HOST \
    -U $SUPER_USER \
    -c "create role $DB_USER with login password 'SET-YOUR-AUTHENTICATOR-PASSWORD';" $DB_NAME
```


## Application Stack

Create the application cloudformation stack. 

Right now we will use `DesiredCount=0` since our application is not deployed yet
(db is empty and the OpenResty images are not uploaded, this will be done by CircleCI)

```sh
aws cloudformation create-stack \
--stack-name $COMPOSE_PROJECT_NAME \
--template-body file://cloudformation/application.yml \
--capabilities CAPABILITY_IAM \
--parameters \
ParameterKey=ClusterName,ParameterValue=$CLUSTER_NAME \
ParameterKey=DesiredCount,ParameterValue=0 \
ParameterKey=Version,ParameterValue=v0.0.0 \
ParameterKey=OpenRestyImage,ParameterValue=$OPENRESTY_REPO_URI \
ParameterKey=ListenerHostNamePattern,ParameterValue=mydomain.com \
ParameterKey=HasHttpsListener,ParameterValue=Yes \
ParameterKey=DbHost,ParameterValue=$PRODUCTION_DB_HOST \
ParameterKey=DbPassword,ParameterValue=SET-YOUR-AUTHENTICATOR-PASSWORD \
ParameterKey=JwtSecret,ParameterValue=SET-YOUR-JWT-SECRET \
```

## Github Repository
Your current application directory is a local git repository, we need to push it to to a remote one (GitHub)

[Create](https://help.github.com/articles/creating-a-new-repository/) an empty GitHub repository.
On the next page, find the section that says **or push an existing repository from the command line**
and execute those command in the root folder of your project. They should look something like this

```bash
git remote add origin https://github.com/<your-github-user>/<repo-name>.git
git push -u origin master
```

Now your code is in GitHub.

## CircleCI build pipeline

The starter kit comes with a CircleCI [configuration file](https://github.com/subzerocloud/postgrest-starter-kit/blob/master/.circleci/config.yml)
that will build your code and push it to production.

- [Signup](https://circleci.com/signup/) to CircleCI using your Github account.
- Add your project's Github repo to the list repos CircleCI will build when new code is pushed.

Once you add your project, CircleCI will immediately start building your project. This is ok, it will not push it to production yet.
If everything went right, the build should finish successfully (green).

- In CircleCI interface, go to the configuration of your project and setup all the env variables listed in the config file [here](https://github.com/subzerocloud/postgrest-starter-kit/blob/master/.circleci/config.yml#L1-L15)

With this last step, your production infrastructure is ready to receive the first version of your application.
