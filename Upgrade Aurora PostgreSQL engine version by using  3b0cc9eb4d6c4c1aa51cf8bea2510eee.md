# Upgrade Aurora PostgreSQL engine version by using blue/green deployments with CDK and AWS CLI

Hello there! We're really excited to help you upgrade your Aurora PostgreSQL engine version. We've got a super neat way of doing this using the Green/Blue deployment strategy. This cool method lets us upgrade smoothly and efficiently with as little downtime as possible.

We'll be using the Cloud Development Kit (CDK) and the AWS Command Line Interface (CLI) to make this happen. Now, we know you might be thinking, "Can't we just use CDK?" Unfortunately, it's not yet natively supported, but we're hopeful for the future!

Here's the fun part: the Blue/Green deployment strategy. We'll create a shiny 'Green' version of your database with the new engine version. After we've made sure it's stable, we'll switch the traffic from the 'Blue' (your old version) to the 'Green' (your new version). It's a safe and secure way of transitioning and provides a nice safety net in case anything unexpected pops up.

In this guide, we are using the AWS Cloud Development Kit (CDK) with Python as our programming language. The CDK is a software development framework to define cloud infrastructure as code and provision it through AWS CloudFormation. Python, being a highly readable and versatile language, allows us to define and deploy our AWS resources efficiently. This combination of CDK and Python gives us the power to automate our cloud infrastructure management and apply software development best practices, such as version control, testing, and CI/CD.

## Motivation

We currently run an Aurora PostgreSQL in RDS with the engine version 12.14. Our goal is to upgrade this engine to version 14.10 while reducing the downtime.

![RDS PostreSQL cluster.](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled.png)

RDS PostreSQL cluster.

## Creating or Updating Your Database Cluster

To begin, create or update your parameter group. This step activates logical replication in Aurora PostgreSQL, which is necessary for running the green/blue deployment.

```python
from aws_cdk import aws_ec2 as ec2
from aws_cdk import Duration
from aws_cdk import aws_rds as rds
from constructs import Construct

class ManagedDatabaseCluster(Construct):
    def __init__(
        self,
        scope: Construct,
        id: str,
        instances: int = 2,
        instance_class: ec2.InstanceClass = ec2.InstanceClass.BURSTABLE3,
        instance_size: ec2.InstanceSize = ec2.InstanceSize.MEDIUM,
        backup_retention_days: int = 30,
        vpc: ec2.Vpc = None,
    ):
        super().__init__(scope, id)

        engine = rds.DatabaseClusterEngine.aurora_postgres(
            version=rds.AuroraPostgresEngineVersion.VER_12_14,
        )

        cluster_paramter_group = rds.ParameterGroup(
            self,
            id="cluster-parameter-group",
            engine=engine,
            parameters={
                "rds.logical_replication": "1",  # Enable logical replication to allow blue/green deployments
                "max_replication_slots": "25",
                "max_wal_senders": "25",
                "max_logical_replication_workers": "25",
                "max_worker_processes": "50",
            },
        )

        self.rds_cluster = rds.DatabaseCluster(
            self,
            "db-cluster",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.of(
                    aurora_postgres_full_version="12.4",
                    aurora_postgres_major_version="12.4",
                ),
            ),
            instance_identifier_base=f"db-cluster",
            instance_props=rds.InstanceProps(
                instance_type=ec2.InstanceType.of(
                    instance_class=instance_class, instance_size=instance_size
                ),
                vpc=vpc,
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_NAT
                ),
            ),
            credentials=rds.Credentials.from_username(
                # your credentails
            ),
            backup=rds.BackupProps(retention=Duration.days(backup_retention_days)),
            cluster_identifier="main-cluster",
            default_database_name="main",
            instances=instances,
            parameter_group=cluster_paramter_group,
            storage_encrypted=True,
        )
```

Next, you need to execute the deployment to create or update your database.

**Note: If you're updating your instance, it will necessitate a reboot of the RDS instances to update the parameters. According to AWS, you might experience a fleeting outage, which can be minimized by reducing traffic.**

*The reboot time for your DB instance depends on the crash recovery process, database activity at the moment of reboot, and your specific DB engine's behavior. To expedite the reboot process, it's recommended to minimize database activity as much as possible. This reduces rollback activity for ongoing transactions.* [AWS Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_RebootInstance.html)

## Creating the Blue/Green deployment

Next, we will create our Blue/Green deployment using the AWS Command Line Interface (CLI). For this guide, we're using version 2.15.14 of the AWS CLI. Before proceeding, please ensure that logical replication has been activated as detailed in the previous steps. This is crucial for the successful execution of the Blue/Green deployment strategy.

You have you run the following command to run the Blue/Green deployment.

```bash
aws rds --region eu-west-1 create-blue-green-deployment --source <source cluster ARN> --target-db-cluster-parameter-group-name default.aurora-postgresql14 --target-engine-version 14.10 --blue-green-deployment-name test-green-blue-deployment
```

Once this is finished you will see 2 databases created.

![Blue and Green databases](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled%201.png)

Blue and Green databases

## Switching to Green instances

In this next step, we are going to switch to our Green database. This is where the magic happens. The 'Green' deployment is the new database version that we created using the Blue/Green deployment strategy. We've already made sure it's stable and ready to go. By running the following command, we will redirect the traffic from the 'Blue' deployment (your old version) to the 'Green' deployment (your new version). This allows us to smoothly transition to the new database version with minimal downtime.

![Blue/Green deployment identifier](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled%202.png)

Blue/Green deployment identifier

```bash
aws rds --region eu-west-1 switchover-blue-green-deployment --blue-green-deployment-identifier bgd-ff7wh1ai80p8yrof
```

![Switching to Green instance](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled%203.png)

Switching to Green instance

## Deleting the Blue/Green deployment

Once the switchover to the 'Green' deployment is successful, we have effectively upgraded to the new database version with minimal disruption to our services. Now, we are going to proceed with removing the 'Blue' deployment, which represents the old version of our database. This is an essential cleanup step that ensures we do not have lingering, outdated resources in our infrastructure.

![Deleting Blue/Green deployment](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled%204.png)

Deleting Blue/Green deployment

Before you jump to the next step and run the CDK, ensure you have deleted the suffix "-old1" from your database instance. This is an important step for successful execution of the CDK.

## Fixing drift in CDK

After deleting the Blue/Green deployment, we might encounter a situation known as "stack drift" in AWS CloudFormation. Stack drift occurs when the actual stack resources differ from the expected configuration values. In this case, the difference lies in the engine version, as we've upgraded it during the Blue/Green deployment. Also, in the cluster parameter name since we haven’t created one and we have used the default.

To rectify this drift, we need to update the engine version in our Cloud Development Kit (CDK) code. CDK uses AWS CloudFormation under the hood, so by updating the engine version in CDK, we ensure that the actual and expected configurations match.

Once we've made this change in our CDK code, we deploy the CDK stack. This action updates the AWS CloudFormation stack, thus syncing the actual and expected configurations and resolving the stack drift. This step is crucial for maintaining the accuracy and reliability of our infrastructure as code.

![Untitled](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled%205.png)

## Fixing CDK drifts

Hey there, to get our CDK drifts back on track, we just need to give our infrastructure a little update to match the current values. It's as simple as tweaking our CDK code to mirror the changes we've made during our Blue/Green deployment - think of the upgraded engine version and the new cluster parameter name. Once we've made those changes in the CDK, we can roll out the updated CDK stack. Doing this brings our actual infrastructure back in line with the expected configuration and clears up those CDK drifts. It's a super important step to ensure our infrastructure as code stays accurate and reliable.

```python
from aws_cdk import aws_ec2 as ec2
from aws_cdk import Duration
from aws_cdk import aws_rds as rds
from constructs import Construct

class ManagedDatabaseCluster(Construct):
    def __init__(
        self,
        scope: Construct,
        id: str,
        instances: int = 2,
        instance_class: ec2.InstanceClass = ec2.InstanceClass.BURSTABLE3,
        instance_size: ec2.InstanceSize = ec2.InstanceSize.MEDIUM,
        backup_retention_days: int = 30,
        vpc: ec2.Vpc = None,
    ):
        super().__init__(scope, id)

        engine = rds.DatabaseClusterEngine.aurora_postgres(
            version=rds.AuroraPostgresEngineVersion.VER_12_14,
        )

        parameter_group = rds.ParameterGroup.from_parameter_group_name(
            self,
            "ParameterGroup",
            "default.aurora-postgresql14",
        )

        self.rds_cluster = rds.DatabaseCluster(
            self,
            "db-cluster",
            engine=rds.DatabaseClusterEngine.aurora_postgres(
                version=rds.AuroraPostgresEngineVersion.of(
                    aurora_postgres_full_version="14.10",
                    aurora_postgres_major_version="14.10",
                ),
            ),
            instance_identifier_base=f"db-cluster",
            instance_props=rds.InstanceProps(
                instance_type=ec2.InstanceType.of(
                    instance_class=instance_class, instance_size=instance_size
                ),
                vpc=vpc,
                vpc_subnets=ec2.SubnetSelection(
                    subnet_type=ec2.SubnetType.PRIVATE_WITH_NAT
                ),
            ),
            credentials=rds.Credentials.from_username(
                # your credentails
            ),
            backup=rds.BackupProps(retention=Duration.days(backup_retention_days)),
            cluster_identifier="main-cluster",
            default_database_name="main",
            instances=instances,
            parameter_group=parameter_group,
            storage_encrypted=True,
        )
```

## Bibliography

- Rümpler, T. (2023). Zero Downtime upgrade of AWS Aurora from MySQL 5.7 to 8.0 with Blue/Green deployments. Retrieved from [https://ruempler.eu/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/](https://ruempler.eu/2023/10/08/zero-downtime-upgrade-aws-aurora-mysql5-7-8-0-with-blue-green/)
- [Creating a Blue/Green deployment for Aurora PostgreSQL DB clusters](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/blue-green-deployments-creating.html#blue-green-deployments-creating-preparing-postgres)
- [Rebooting a DB instance](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_RebootCluster.html#aurora-reboot-db-instance)

![Untitled](Upgrade%20Aurora%20PostgreSQL%20engine%20version%20by%20using%20%203b0cc9eb4d6c4c1aa51cf8bea2510eee/Untitled%206.png)