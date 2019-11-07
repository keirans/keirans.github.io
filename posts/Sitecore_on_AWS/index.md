---
layout: post
---
## Introduction

Sitecore is one of the most popular and feature rich Content Management Systems (CMS) out there today. Mostly deployed in more traditional on-premise topologies, I recently was engaged to work with a client to install and configure it on AWS. 

My  goal was to offload as much of Sitecore's dependancies off to native AWS services to keep the automation levels high, the management levels low and to represent as much of the solution as code as possible.

Sitecore doesn't officially support all these AWS services, so using them is at your risk and discretion, however the following notes are my way of giving back to the Sitecore community who's blog posts, presentations and code snippets helped me extensively when working on this project.

It is worth noting that Sitecore's software stack is currently being enhanced and I'd expect to see some of these steps become eventually obsoleted in public cloud environments as they move to container based deployment options, resulting in less EC2 instances being required (yay).


## Baking suitable Windows AMI's for use as Sitecore Servers

At it's core, Sitecore today is a Windows application with its main services implemented as either Windows web applications or as Windows services.

As of Sitecore 9.x the application is installed across a number of servers using a Powershell module called the Sitecore Installation Framework or SIF.

As I needed to deploy a number of Windows servers of which Sitecore's various roles were to be installed on, I opted to use [Packer](https://packer.io) to create an AMI that had all the various dependancies required for installation in place. In the below example code, you can see my high level Packer template and some powershell snippets that install all the various dependancies automatically.

A couple of important things to note is that you _*must*_ sysprep your instance before baking the AMI, the Sitecore documentation details that servers behind load balancers must have different identitites. In addition to this, the installation of the Sitecore prereq tasks must use Packers elevated_user option to run as Administrator, failure to do so results in SIF not succeeding. You can see these configuration snippets in my example code below.


### Packer Example configuration

The below Packer configuration will be enough to get you running. Add your network definitions, security groups, instance policies and AMI's as required.

I built on Windows 2016 Core and once the AMIs were ready, deployed them using Cloudformation with the associated Load Balancers and other resources.

```
{
  "builders": [
    {
      "type": "amazon-ebs",
      "region": "ap-southeast-2",
      "encrypt_boot": true,
      "instance_type": "t3.large",
      "source_ami" : "ami-idgoeshere",
      "ami_name": "Sitecore-Windows2016-{{timestamp}}",
      "iam_instance_profile" : "Instance-Profile-Here",
      "user_data_file":  "./userdata.txt",
      "communicator": "winrm",
      "winrm_username": "Administrator",
      "winrm_insecure": true,
      "winrm_use_ssl": false,
    }
  ],
  "provisioners": [
    {
      "script": "sitecore-bootstrap.ps1",
      "type": "powershell",
      "elevated_user": "Administrator",
      "elevated_password": "{{.WinRMPassword}}",
    },
    {
      "type": "windows-restart"
    },
    {
      "type": "powershell",
      "inline": [
      "C:\\ProgramData\\Amazon\\EC2-Windows\\Launch\\Scripts\\SendWindowsIsReady.ps1 -Schedule",
      "C:\\ProgramData\\Amazon\\EC2-Windows\\Launch\\Scripts\\InitializeInstance.ps1 -Schedule",
      "C:\\ProgramData\\Amazon\\EC2-Windows\\Launch\\Scripts\\SysprepInstance.ps1 -NoShutdown"
    ]
  }]
}

```

### Provisioning script powershell logic example

Within the provisioning script (sitecore-bootstrap.ps1) executed by Packer I take the following actions:

* Copy the Sitecore installation media ZIP file down from S3 to the instance
* Unpack the ZIP into a temp directory
* Install the Sitecore Installation Framework (SIF) powershell modules onto the host
* Run the Install-SitecoreConfiguration cmdlet from SIF passing in the Prerequisites.json configuration file, this will orchestrate the installation of the many dependancies that Sitecore has before you can even start installing the other roles. (There are gigabytes of them)
* Exit and hand back to Packer for a reboot and SysPrep of the instance.

Some of the sample Powershell code is below for your reference;

```
Write-Host "Extracting the Sitecore XP1 Configuration"
Write-Host "====================================================================="
Expand-Archive "XP1 Configuration files 9.1.1 rev. 002459.zip" -DestinationPath Config
cd "C:\windows\temp\sitecore\Config"

# Install SIF
Write-Host "Installing the Sitecore Installation Framework"
Write-Host "====================================================================="
Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force
Register-PSRepository -Name SitecoreGallery -SourceLocation https://sitecore.myget.org/F/sc-powershell/api/v2 -InstallationPolicy Trusted
Install-Module -Name SitecoreInstallFramework -Repository SitecoreGallery

# Install Pre-Reqs
Write-Host "Running the Sitecore SIF Prereqs installer"
Write-Host "====================================================================="
mkdir C:\Windows\Temp\Sitecore-Temp-Downloads
Install-SitecoreConfiguration -Path "C:\windows\temp\sitecore\Config\Prerequisites.json" -Verbose -TempLocation "C:\Windows\Temp\Sitecore-Temp-Downloads"
```

## Preparing for RDS Installation

Sitecore stores all of it's state in a set of Databases. The database platform that we opted to use was MS SQL Server 2016 SP2. This was well supported by the Vendor, the clients team and also RDS.

To get things working, we needed to work through a few issues.

### Customising the Sitecore Installation Media to Support RDS installation

When installing Sitecore using SIF it creates a large number of additional databases for all of it's various roles. To achieve this, it executes a set of SQL code that is in the install media.

A number of these SQL statements are not compatible with RDS as they execute statements like the following;

```
sp_configure 'contained database authentication', 1;
GO
RECONFIGURE;
GO
```

As such, you need to take all of the Sitecore web deploy packages from the installation media, unpack them and remove all the offending SQL statements and re-zip them.

This is covered well at the following Sitecore blog post [Deploying Sitecore 9 in AWS RDS](https://jeroen-de-groot.com/2018/07/19/deploying-sitecore-9-in-aws-rds/) and you will have to repeat it for all the web deploy packages you use that contain the offending SQL logic.

### Deploying your RDS Database with AWS Cloudformation

Now that we have removed the offending SQL from the installation media, we need to configure the required settings the SQL was trying to implement the RDS sanctioned way, using the DBParameterGroup.

In the below cloudformation example, you can see the RDS Database is created and associated with the DBParameterGroup suitable for Sitecore with "contained database authentication" set accordingly.

Once this is in place, Sitecore will install accordingly.

Please note that the initial RDS Database should be configured as a Single AZ initially for install, more on this next.

```
AWSTemplateFormatVersion: '2010-09-09'

Description: Sitecore RDS Instance stack

Resources:

  RDSInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot
    Properties:
      DeletionProtection: !Ref DeletionProtection
      AllocatedStorage: !Ref AllocatedStorage
      BackupRetentionPeriod: !Ref BackupRetentionPeriod
      CopyTagsToSnapshot: true
      DBInstanceClass: !Ref InstanceType
      DBInstanceIdentifier: !Ref DBInstanceIdentifier
      DBSubnetGroupName: !Ref DBSubnetGroupName
      DBParameterGroupName: !Ref RDSParamGroup
      DeleteAutomatedBackups: false
      Engine: sqlserver-se
      EngineVersion: 13.00.5216.0.v1
      KmsKeyId: !Ref KmsKeyId
      MasterUserPassword: !Ref MasterUserPassword
      MasterUsername: !Ref MasterUsername
      LicenseModel: license-included
      MultiAZ: False
      PubliclyAccessible: false
      StorageEncrypted: true
      StorageType: gp2
      VPCSecurityGroups:
        - !Ref SecurityGroup

  RDSParamGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties: 
      Description: Sitecore-Param-Group
      Family: sqlserver-se-13.0
      Parameters: 
        "contained database authentication" : 1

```


### Multi-AZ Challenges

When building our production environment we deployed our RDS Instance with Multi-AZ True as we wanted to ensure that we had high availiability in the event of an instance of AZ failure, however when we ran the Sitecore installer, we got the following error.

```
Error SQL72014: .Net SqlClient Data Provider: Msg 5069, Level 16, State 1, Line 5 ALTER DATABASE statement failed.
Error SQL72045: Script execution error.  The executed script:
IF EXISTS (SELECT 1
           FROM   [master].[dbo].[sysdatabases]
           WHERE  [name] = N'$(DatabaseName)')
    BEGIN
        ALTER DATABASE [$(DatabaseName)]
            SET PAGE_VERIFY NONE,
                DISABLE_BROKER 
            WITH ROLLBACK IMMEDIATE;
    END


 Microsoft.SqlServer.Dac.DacServicesException: Could not deploy package.
Error SQL72014: .Net SqlClient Data Provider: Msg 9778, Level 16, State 1, Line 5 Cannot create a new Service Broker in a mirrored database "Sitecore_Xdb.Collection.Shard0".
Error SQL72045: Script execution error.  The executed script:
IF EXISTS (SELECT 1
           FROM   [master].[dbo].[sysdatabases]
           WHERE  [name] = N'$(DatabaseName)')
    BEGIN
        ALTER DATABASE [$(DatabaseName)]
            SET PAGE_VERIFY NONE,
                DISABLE_BROKER 
            WITH ROLLBACK IMMEDIATE;
    END
```

Digging a little deeper we found that Sitecore doesnt support the installation on a mirrored database. Working with the team we opted to do the install on a Single-AZ RDS instance and then enable Muilti-AZ after.

We found this worked OK and resolved our issues, with the internal Sitecore team giving it a greenlight.

### Validating the mirroring of your RDS Databases once you have Multi-AZ in place

After we turned on the Multi-AZ, I needed to have piece of mind that the mirroring was still functional behind the scenes with RDS because I knew that Sitecore had created a large number of additional databases on the instance, and they had to be mirrored to the other AZ as well.

Speaking to AWS Support, they provided me with some SQL Code that I wrapped in Powershell and executed via SSM Session manager on one of our Windows Management hosts.

This provides visibility of the mirroring state on the RDS Instance for each database present, as you can see there are a number of Databases and they are all in good condition mirroring wise.


```

$command = @'
SELECT
DB_NAME(database_id) As DatabaseName,
CASE WHEN mirroring_guid IS NOT NULL THEN 'Mirroring is On' ELSE 'No mirror configured' END AS IsMirrorOn,
mirroring_state_desc,
CASE WHEN mirroring_safety_level=1 THEN 'High Performance' WHEN mirroring_safety_level=2 THEN 'High Safety' ELSE NULL END AS MirrorSafety,
mirroring_role_desc,
mirroring_partner_instance AS MirrorServer
FROM sys.database_mirroring where database_id>4
GO
'@

PS C:\Windows\system32> Invoke-Sqlcmd $command  -ServerInstance <RDS DNS NAME> -Username sa -Password <PASSWORD>


DatabaseName         : rdsadmin
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_MarketingAutomation
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Messaging
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Processing.Pools
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Xdb.Collection.ShardMapManager
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Xdb.Collection.Shard0
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Xdb.Collection.Shard1
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_ReferenceData
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_ProcessingEngineTasks
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_ProcessingEngineStorage
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Reporting
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Core
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Master
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Web
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_ExperienceForms
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_EXM.Master
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : Sitecore_Processing.Tasks
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

DatabaseName         : GlobalLink
IsMirrorOn           : Mirroring is On
mirroring_state_desc : SYNCHRONIZED
MirrorSafety         : High Safety
mirroring_role_desc  : PRINCIPAL
MirrorServer         : Mirror-ServerName

PS C:\Windows\system32>

```


## Offloading Session state to AWS Elasticache Redis

The last thing I looked into was offloading the Session state to Elasticache, Redis is already well supported in Sitecore for this, however it was a bit light on details. After a bit of digging, I discovered the only version supported currently is 3.2.6 and this can be easily deployed using Cloudformation in a Relication group using the below CloudFormation resources.

Once you get your endpoint details, you can update your Sitecore configuration with them accordingly.

```
AWSTemplateFormatVersion: '2010-09-09'

Description: Sitecore Multi-AZ Redis Cache for Session management

Resources:

  ReplicationGroup:
    Type: 'AWS::ElastiCache::ReplicationGroup'
    Properties:
      CacheParameterGroupName: !Ref CacheParams
      AtRestEncryptionEnabled: True
      AutomaticFailoverEnabled: !Ref MultiAZSupport
      CacheNodeType: !Ref CacheNodeType
      CacheSubnetGroupName: !Ref CacheSubnetGroupName
      Engine: "redis"
      EngineVersion: 3.2.6
      NumCacheClusters: !Ref NumCacheClusters
      ReplicationGroupId: !Ref ReplicationGroupId
      ReplicationGroupDescription: !Ref 
      SecurityGroupIds:
        - !Ref SecurityGroupIds

  CacheParams:
    Type: AWS::ElastiCache::ParameterGroup
    Properties: 
      CacheParameterGroupFamily: redis3.2 
      Description: SitecoreCacheParams
```

## Wrapping up

As you can see, with a bit of effort and debugging, it is possible to offload a number of Sitecores functionality to native AWS services rather than have to go through the effort of building and maintaining them yourself.

