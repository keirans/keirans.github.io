---
layout: post
author: Keiran
title: RDS Database exports to S3 for Sitecore
---

## Introduction
[In a previous blog post](https://notes.keiran.io/posts/Running_Sitecore_on_AWS/) I detailed how we were able to get Sitecore working with RDS for SQL server. However one challenge did come up, specifically in regards to enabling the application team to get copies of the various databases stored within RDS for use in other environments, to provide copies to vendor support teams or to take individual database backups prior to upgrades where a whole instance snapshot would be excessive.

[Fortunately AWS provide a set of stored procedures](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.Procedural.Importing.html) that can be used to enable application teams to invoke individual database exports that can then be retreived from S3 directly for use elsewhere. In this post, I'll provide the cloudformation code to setup RDS to provide this level of funcitonality, as I found examples were limited.

### Creating the Database exports bucket
The first step is to create an S3 Bucket for the Database backups to be placed in by the RDS Service.

In this example, I also attach a 14 day lifecycle policy as I don't want to have as we don't want to have database exports sitting around in S3 longer than that, as well as enabling AES256 encryption on the server side for foundational security.

```
  SitecoreDBBackupsBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Ref BucketName
      LifecycleConfiguration:
        Rules:
          - Id: SitecoreDBBackupsLifecyclePolicy
            ExpirationInDays: 14 
            Status: Enabled
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: AES256
```

### Creating the RDS Role for access to the S3 Bucket
Next up, we create an IAM role for the RDS database that has a policy allowing the RDS service to assume it, thus inheriting it's access to the resources within our account, as well as a set of policies that grant the RDS service access to put the export data into it.

You will see that we explicitly define the buckets that this applies to using it's ARN. In your code you will have to put the ARN of the bucket created in the above step, either by using a Ref if it is in the same stack, or using the [Cloudformation Import/Export](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-exports.html) functionality if it resides in another stack.


```
  RdsHostRole: 
    Type: "AWS::IAM::Role"
    Properties:
      AssumeRolePolicyDocument: 
        Version: "2012-10-17"
        Statement: 
          - 
            Effect: "Allow"
            Principal: 
              Service: 
                - "rds.amazonaws.com"
            Action: 
              - "sts:AssumeRole"
      Path: "/"
      Policies: 
        - 
          PolicyName: "RDSBackups"
          PolicyDocument: 
            Version: "2012-10-17"
            Statement: 
              - 
                Effect: "Allow"
                Action: 
                  - "s3:ListBucket"
                  - "s3:GetBucketLocation"
                Resource:
                  - arn:aws:s3:::bucketname-here
              - 
                Effect: "Allow"
                Action: 
                  - "s3:GetObject"
                  - "s3:PutObject"
                  - "s3:ListMultipartUploadParts"
                  - "s3:AbortMultipartUpload"
                Resource:
                  - arn:aws:s3:::bucketname-here/*

```

### Configuring RDS to use the Bucket and IAM role
Finally, we create our MS SQL RDS Instance that will house our Sitecore data.

Looking at the resources below, you will notice the following:

#### AWS Log Groups
RDS for MS SQL Server now supports exporting the agent and error logs to Cloudwatch logs. This is a great feature that allows us to pull this data easily into external logging systems, however if you enable it with the Cloudformation EnableCloudwatchLogsExports parameter, it will create the log groups for you with an unlimited retention period and log groups that are not managed and lifecycled with the RDS instance iself, leaving potentially costly unmanaged log data in your account.

Because of this, I create 3 log groups before I create the database, one for the agent logs, one for the error logs and then one for rds-events that is only ever created for multi-AZ instances.

To ensure that I create the log groups with the correct names, I pass in 4 Parameters into the template: Application, Branch, Build and Environment and I use this to build the log group name and the DBInstanceIdentifier value on the RDS Instance itself. As long as the log group is present before the database, RDS will start logging to it.

#### RDS RDSParamGroup

[The RDS Param Group settings for Sitecore is detailed in a previous blog post](https://notes.keiran.io/posts/Running_Sitecore_on_AWS/) and you can find more about these settings there if required.

#### RDSOptionGroup
The RDS Option group is a new resource added to our Sitecore database stack, defining and associating it with the RDS Instance ties everything together, specifically, it enables the SQLSERVER_BACKUP_RESTORE option on the RDS instance, but also it associates the IAM role that we have created in the previous steps that provide it access to the S3 bucket to store the exported data.

As you can see from the Option Group resource, you will need to update the template with the Role ARN in the place of _<ARN OF RDS ROLE FROM PREVIOUS STEP>_ either using a Ref or an Import depending on how you have laid out your templates and stacks.


#### RDSInstance
The RDS Instance is then created accordingly for use, note the DBParameterGroupName , OptionGroupName and EnableCloudwatchLogsExports properties that are detailed above being applied accordingly.


```
  AgentLogGroup:
    Type: AWS::Logs::LogGroup
    Properties: 
      LogGroupName: !Sub "/aws/rds/instance/${Application}-${Branch}-${Build}-${Environment}/agent"
      RetentionInDays: !Ref RetentionInDays

  ErrorLogGroup:
    Type: AWS::Logs::LogGroup
    Properties: 
      LogGroupName: !Sub "/aws/rds/instance/${Application}-${Branch}-${Build}-${Environment}/error"
      RetentionInDays: !Ref RetentionInDays

  # Multi-AZ Only
  EventsLogGroup:
    Type: AWS::Logs::LogGroup
    Properties: 
      LogGroupName: !Sub "/aws/rds/instance/${Application}-${Branch}-${Build}-${Environment}/rds-events"
      RetentionInDays: !Ref RetentionInDays

  RDSInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DBInstanceIdentifier: !Sub "${Application}-params-${Branch}-${Build}-${Environment}"
      AllocatedStorage: !Ref AllocatedStorage
      BackupRetentionPeriod: !Ref BackupRetentionPeriod
      DBInstanceClass: !Ref InstanceType
      DBSubnetGroupName: !Ref DBSubnetGroupName
      EnableCloudwatchLogsExports:
        - error
        - agent
      DBParameterGroupName: !Ref RDSParamGroup
      DeleteAutomatedBackups: false
      Engine: sqlserver-se
      EngineVersion: 13.00.5216.0.v1
      KmsKeyId: !Ref KmsKeyId
      MasterUserPassword: !Ref MasterUserPassword
      MasterUsername: !Ref MasterUsername
      LicenseModel: license-included
      MultiAZ: !Ref MultiAZ
      PubliclyAccessible: false
      StorageEncrypted: true
      StorageType: gp2
      OptionGroupName: !Ref RDSOptionGroup
      VPCSecurityGroups:
        - !Ref SecurityGroup

  RDSParamGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties: 
      Description: !Sub "${Application}-params-${Branch}-${Build}-${Environment}"
      Family: sqlserver-se-13.0
      Parameters: 
        "contained database authentication" : 1

  RDSOptionGroup:
    Type: AWS::RDS::OptionGroup
    Properties: 
      OptionGroupDescription: !Sub "${Application}-options-${Branch}-${Build}-${Environment}"
      EngineName: sqlserver-se
      MajorEngineVersion: "13.00"
      OptionConfigurations:
        - OptionName: SQLSERVER_BACKUP_RESTORE
          OptionSettings:
            - Name: IAM_ROLE_ARN
              Value: <ARN OF RDS ROLE FROM PREVIOUS STEP>
```

### Invoking the database export with powershell
Once the instance is up and running, we can now invoke the stored procedures that are exposed by RDS to take an export of an individual database within the RDS instance and have it stored in S3 for use.

In the below example, I have provided some Powershell that uses Invoke-Sqlcmd to export the Sitecore_Core database to our S3 bucket as a file called Sitecore_Core-2019-11-24.bak.

```
PS C:\Windows\system32> $command = @'
>> exec msdb.dbo.rds_backup_database
>> @source_db_name='Sitecore_Core',
>> @s3_arn_to_backup_to='arn:aws:s3:::bucketname-here/Sitecore_Core-2019-11-24.bak',
>> @overwrite_s3_backup_file=1,
>> @type='FULL';
>> '@
PS C:\Windows\system32> Invoke-Sqlcmd $command  -ServerInstance <RDS DNS NAME> -Username sa -Password 


task_id                  : 12
task_type                : BACKUP_DB
lifecycle                : CREATED
created_at               : 11/24/2019 4:19:25 AM
last_updated             : 11/24/2019 4:19:25 AM
database_name            : Sitecore_Core
S3_object_arn            : arn:aws:s3:::bucketname-here/Sitecore_Core-2019-11-24.bak
overwrite_S3_backup_file : True
KMS_master_key_arn       :
task_progress            : 0
task_info                :


PS C:\Windows\system32>
```

Using powershell in this way means that it can easily be done using AWS System Manager session manager to a Windows Management system that has access to the RDS ports, thus not having to use RDP.

For this to function in your environment, it's important to ensure that you:
* Are running this on an Instance that has access to your RDS Database port
* Update your S3 Bucket ARN accordingly so it is targeting your own bucket


### KMS Encrypted backups and decrypting them client side
Please be aware that databases exported using the above process are not encrypted, and you should be leveraging the KMS encryption functionality of this capability, resulting in KMS encrypted database exports.

When this is done, any database exports that are residing in S3 must be decrypted after download before they can be used.

This process is well documented at the below blog posts and it is suggested that it is used.

* [Importing and Exporting SQL Server Databases](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/SQLServer.Procedural.Importing.html)
* [Client-Side Encryption and Decryption of Microsoft SQL Server Backups for Use with Amazon RDS](https://aws.amazon.com/blogs/database/client-side-encryption-and-decryption-of-microsoft-sql-server-backups-for-use-with-amazon-rds/)

