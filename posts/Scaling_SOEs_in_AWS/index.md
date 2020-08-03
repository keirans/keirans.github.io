---
layout: post
author: Keiran
title: Scaling Standard Operating Environments in AWS 
---

# Introduction to SOEs

A Standard Operating Environment (SOE) implements and standardizes an operating system and its associated software, resulting in the reduction in the cost and time taken to deploy, configure, maintain, support, secure and manage the operating systems across a large fleet of virtual machine instances.

In AWS environments, SOEs are distributed in the form of [Amazon Machine Images (AMIs)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AMIs.html). It is not uncommon for environments of all sizes to have multiple SOE flavors that cover different Operating System types and releases, such as Microsoft Windows and various distributions of Linux and the subversions of each.

In environments that follow the [immutable infrastructure model](https://cloudacademy.com/course/advanced-deployment-techniques-on-aws/immutable-infrastructure-2/), the SOE is released on monthly or similar cadence providing new features, capabilities, security patches and more, with mid-cycle releases often occuring to cater for emergency security patches and fixes.

Once updated SOEs are produced, it then is up to downstream application teams to adopt the latest SOE as part of their continuous delivery practices and re-deploy their applications ontop of it.

In the many AWS environments I've worked in over the years, I have observed organisations that have in excess of 10+ unique SOEs that need to be distributed to tens, of not hundreds of AWS accounts for use by a wide variety of application teams.

In this post, I'll detail some of the learnings I have had over the years to ensure that when building SOEs in AWS you produce a high quality product for consumption by your users.

# Pre-Development steps to success
## Gather clear requirements and building solid process

Before jumping into the fun stuff of writing code and building AMI's for use, the most important thing that you need to think about is the requirements you are working to and the processes you are going to implement and follow to ensure that you are always producing a high quality product for all users.

When speaking to clients about SOEs, I always emphasise the need to treat each SOE as a software product that needs to align to common development standards, including:

- What is this image going to be used for ? 
  - Will it be used by Immutable or Long Lived Instances and what is the required release cadence for each ?
  - What are the types of applications it will be hosting and what packages and configuration settings will it need to have to support them ?
  - How will the end user consume this SOE (ie, Terraform Modules, Cloudformation Templates, Service Catalogue) ?

- How will end users discover and use this AMI ? 
  - Where will the user guide be to ensure that it's clear and easy to use ? 
  - How will end users and their applications programatically discover the latest AMI's from their automation pipelines ?
  - How will the release notes be distributed with each release so that changes are well understood ?
  - How will end users and maintainers track enhancements and defects to ensure they are addressed and delivered clearly ?

In the case of release notes and issue tracking, it is my opinion that all SOE related tasks are done so in JIRA (or a similar tracking tool), and release notes are automatically generated, reducing manual work for each release.

## Buy or Build - Evaluating the impact of using marketplace AMI's
Many organisations often opt to use pre-configured AMI's from the AWS Marketplace to get up and running faster, using these AMI's as the basis for their SOEs instead of the vanilla ones from Microsoft, Redhat or Ubuntu.

Although these can help, you should consider the following potential implications before use to ensure that they don't come with unforseen consequences.

Some things to think about are;

* Many marketplace AMI's have additional costs per hour. Initially a few cents in the dollar per hour may not seem like much, however when you are running thousands of EC2 instances this can add up to thousands of dollars very quickly.

* Not all marketplace AMI's support sub-hour billing granularity. If you have a workload that may benefit for per second billing, make sure you aren't basing your SOE off an image that doesn't support it.

* Who is going to support the AMI if you find issues with it in the future, and what is their SLA for defect resolution ?

* What are the implications to your environment if a marketplace AMI is deprecicated or pulled from the marketplace suddenly ? I've encountered SOEs not being available in newly created AWS accounts because the Source AMI it is built from has been depreciated from the vendor.


## Start small and increment

As the [AWS Well Architected framework emphasises](https://d1.awsstatic.com/whitepapers/architecture/AWS_Well-Architected_Framework.pdf), you should look to  "Make frequent, small, reversible changes". This also applies to your SOE AMI's.

It is a much more effective approach to building AMI's to take a base image and implement the features and capabilities in a gradual and phased approach to ensure that each peice of code is linked to a particular feature or configuration request.

One common problem I have regularly observed is teams taking large hardening and configuration scripts from on-premise systems and applying them to their AWS SOEs hoping that they will provide similar security or configuration outcomes. In most cases, this results in the AMI's become polluted with settings that are not applicable to EC2 instances, or breaking functionality required by services such as EC2Launch.

In addition to this, the AMI's often grow needlessly in size, resulting in additional cost.

## Use a deployment pipeline and automate the entire process

Infrastructure as code and CICD processes are known to be the best way to deliver consistent and repeatable outcomes when working with cloud platforms such as AWS.

There are a wealth of tools out there to assist, with the below being some of the most popular.

* [Hashicorp Packer](https://www.packer.io/)
* [AWS Image Builder](https://aws.amazon.com/image-builder/)

For each SOE you build however, you should look to be able to:
* Store each script, configuration file and other piece of code used to build it in version control (git)
* Track each built AMI to the repository, commit ID and CI task that produced it
* Ensure that a single line of code changed in the SOE repository can build and distribute a AMI without human interaction
* Ensure each change made to the SOE has a corrosponding JIRA or Issue ID

If the concepts of CICD in an Infrastructure context is new to you, I encourage you to have a read of the following documents that will get you across the concepts.

* [The Release Pipeline Model - Microsoft](http://download.microsoft.com/download/C/4/A/C4A14099-FEA4-4CB3-8A8F-A0C2BE5A1219/The%20Release%20Pipeline%20Model.pdf)
* [AWS Well Architected Framework - Specifically,  "Perform operations as code"](https://d1.awsstatic.com/whitepapers/architecture/AWS_Well-Architected_Framework.pdf)


# Implementation Do's and Dont's
After building and working on AWS SOEs for quite some time, here are some tips from the trenches to help you build better AMI's for your organisations.


## Don't disable SELinux
* [It's 2020](https://stopdisablingselinux.com/), and rolling it out to your fleet after you have turned it off is not a very fun exersize.

## Use a Configuration management tool
Configuration management tools shine by helping you configure virtual machines in a delcarative way, resulting in a reduction in large complex bash and powershell scripts.

In addition to this, many modern configuration management tools such as Puppet, Chef and Ansible provide you with additional functionality such as

* Easily handling differences between Operating System types and versions for common tasks like
  * Software and Package Management (apt, rpm, yum, deb, msi, , gem, pip, exe)
  * User and Group Management
  * File distribution and permission management
* Generating configuration files from templates
* Providing a library of modules and example code to get you moving faster
* Out of the box integration with Image baking tools such as Packer

A common misconception I have encountered in my travels is that Windows does not lend itself well to configuration management tools, however products such as Chef and Puppet have mature Windows support with the [later having an extensive library of Windows modules for use.](https://forge.puppet.com/puppetlabs/windows)


## Declare your Linux user / group ID's and keep them consistent
A common problem that harks back to the days of more traditional system administration practices is when Linux User/Group ID numbers change between SOE releases or virtual machine instances.

This becomes a signifigant problem with solutions that use shared filesystems such as EFS or leverage EBS snapshots, as changes in the User and Group ID number(s) can result in files and directories being owned by different users than expected, often to the detriment of the applications on the instance.

The best solution to this is to ensure that all software that is installed that creates user accounts have an ID registered for them in a central database, then in the SOE code you should explicitly create these users / groups with their assigned ID. Where possible, keep these UID's consistent across all your Linux distributions.

In the case of accounts and groups for humans, it's generally best to not place these identities in the SOE, but to integrate the instance with an external directory service such as LDAP or Active Directory.


## Use Linux .d directories 
The Linux operating system has hundreds of configuration files, many of which need settings changed to support the requirements of the application teams and business units.

Updating configuration files in place is not often easy as their formats can vary, and although there are tools such as [Augeas](https://augeas.net/) to help with doing this in an [idempotent way](http://scottpecnik.com/2017/10/23/puppet-idempotency-what-and-why/), configuration file changes often trend back to unmanageable sed and grep commands run over configuration files in loops.

Fortunately, many services configuration files in Linux provide a .d directory framework that lets you drop off complimentary configuration files which are then evaluated in conjunction with the main configuration file, resulting in there being no need to edit large complex configuration files in place.

This approach to configuring services also lends itself well to configuration management tools which excel at templating and distributing small files.

Some great articles about this functionality of the Linux OS can be found below

* [The Linux .d Approach](http://blog.siphos.be/2013/05/the-linux-d-approach/)
* [etc configuration directories](https://www.redhat.com/sysadmin/etc-configuration-directories)


## Rename the Administrator account with caution
Many Security hardening guides (such as CIS) advocate the renaming of the Administrator account to something else to lessen the risk of brute force attacks.

Prior to the release of [EC2Launch v2](https://docs.aws.amazon.com/AWSEC2/latest/WindowsGuide/ec2launch-v2-overview.html), this broke EC2Launch preventing the ["Get Windows Password"](https://aws.amazon.com/premiumsupport/knowledge-center/retrieve-windows-admin-password/) functionality in the AWS console, as well as tools such as Packer that used the corrosponding API call.

As a result, I personally recommend avoiding this process, protecting the Administrator account with a strong password and other measures. If you, in combination with your security team decide to go down this path, I encourage you to thoroughly test all other agents, services and pieces of software that may be impacted by this change.

## Externalise your passwords and runtime params
When a SOE AMI is used to launch a EC2 Instance, a number of actions are invoked through the instance user data or launch scripts baked into the AMI itself.

These actions often include tasks such as:
* Starting services suchs as monitoring tools and security agents
* Joining the instance to Active Directory for user authentication and group policy management
* Integrating the instances into downstream services such as configuration management tools or software repositories

Many of these steps require the instance to have access to passwords or run time configuration information to succeed. When building SOE AMI's it is best to not hardcode these values into the image itself, but develop bootstrap logic that sources this from an external source such as AWS SSM Parameter Store or AWS Secrets manager.

Taking this approach will ensure that the SOE is free from credentials as well as portable across environments without the need for producing additional image updates which can be a timely exersize as your environment grows.

When externalising these runtime values it is also important to ensure you build resilience into the scripts that fetch them via retry and backoff logic if they are not immediately available, or to gracefully handle transient issues in the environment.


## Use Powershell to it's potential across all your Windows versions

[Powershell](https://docs.microsoft.com/en-us/powershell/scripting/overview?view=powershell-7) is an incredibly powerful tool for AWS and Windows Administrators alike. Unfortunately depending on the version of Powershell you have and the capabilities it provides greatly vary between Windows releases, often resulting in Powershell scripts being written that leverage the lowest common denominator feature or capability that is available.

But not all is lost !

Microsoft provides the "[Windows Management Framework](https://docs.microsoft.com/en-us/powershell/scripting/windows-powershell/wmf/overview?view=powershell-7)" or WMF. When installed on earlier Windows, it brings the capabilities of Powershell and its associated tooling up to the same level as those provided in the latest releases.

WMF installation adds and/or updates the following features:

* Windows PowerShell
* Windows PowerShell Desired State Configuration (DSC)
* Windows PowerShell Integrated Script Environment (ISE)
* Windows Remote Management (WinRM)
* Windows Management Instrumentation (WMI)
* Windows PowerShell Web Services (Management OData IIS Extension)
* Software Inventory Logging (SIL)
* Server Manager CIM Provider

When building Windows SOEs that are using pre-2016 versions, install WMF and enjoy simplified code and and Administration processes.

## Set your timezone as UTC 
Correlating log events or scheduling tasks across timezones isn't easy, especially if each system is configured to use different regional timezones.

Because of this, it's best to set all your systems to use the UTC Timezone, and then convert them where required in your application configuration or Linux profile via use of the $TZ variable.

You will find that this correlates with AWS's approach to logging and [scheduling](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) where UTC is also used.

As other people have mentioned in the past, this approach should also be applied to [RDS and other services](https://twitter.com/QuinnyPig/status/997178792257830912)

## Store build time data for later reference and debugging
When you have produced and distributed a number of SOE AMI's in an automated fashion it is often useful to store all the data produced when building the image in a centralised location for debugging purposes in the future. Even the smallest logfile entry overlooked weeks before can guide you to a root cause when raised by an end user.

In previous environments, I've had success with the following approach

- Ship all build logs into AWS Cloudwatch Logs or into a Centralised S3 Bucket
- Create an [SOS report](https://access.redhat.com/solutions/3592) of your systems after your bake process or during automated testing
- Where possible leverage the [Artefact functionality](https://confluence.atlassian.com/bamboo/configuring-a-job-s-build-artifacts-289277071.html) of your CI tool to link to them for easy discovery

# Automated Testing

As SOEs get more features they become more complex and have a higher chance of defects.  Compounding the issue, as you release more SOE variants, this increases your chances once again.

Some of these defects can be simple such as

* Incorrect permissions on the AWS CLI preventing execution by a certain set of users
* A security software package upgrade not ensuring that the service is set to running in systemd
* A spelling mistake in a Windows domain join script resulting in the instance failing to do so
* The sticky bit missing on /tmp causing file permission issues

To more complex, such as 

* EBS volumes being assigned unexpected block device files between EC2 Instance types
* EC2 Instances launched in larger autoscale groups failing to retreive credentials in time from downstream services due to race conditions
* Permissions change on directories by group policy preventing agents from starting as expected

As a result, you should build an automated testing framework early in your SOE development process, and ensure that this testing suite evolves alongside your features providing increased confidence that your SOE works reliably as well as providing a set of gates that must be passed prior to release to your users.

In this section, I will walk through the high level mechanics of building an automated SOE testing framework

## Select a  testing tool

The first thing you need is a suitable testing tool to write your tests in. There are a number of different tools available out there that provide a wide variety of capabilites,  I've encountered the following tools out in the wild and encourage you to start here.

* [ServerSpec](https://serverspec.org/)
  * "Serverspec tests your servers’ actual state by executing command locally, via SSH, via WinRM, via Docker API and so on. So you don’t need to install any agent softwares on your servers and can use any configuration management tools, Puppet, Ansible, CFEngine, Itamae and so on."
* [Inspec](https://community.chef.io/products/chef-inspec/)
  * "Turn your compliance, security, and other policy requirements into automated tests."
* [Pester](https://github.com/pester/Pester)
  * "Pester is the ubiquitous test and mock framework for PowerShell."
* [Goss](https://github.com/aelsabbahy/goss)
  * "Quick and Easy server validation"

## Write your first tests

Once you have selected your tool, you need to start going through your SOEs core functionality and start writing tests for them. 

I've found that simple tests for the presence of software packages or file permissions are a good starting point, and when your confidence grows, add more complex testing scenarios into the mix.

For example, the following ServerSpec test ensures that the AWS CLI is installed and configured correctly. 

When the test is run using ServerSpec, a successful test will result in a zero exit code, and a failure will return non-zero.


```
  describe file('/usr/bin/aws') do
      it { should exist }
      it { should be_file }
      it { should be_owned_by 'root' }
      it { should be_grouped_into 'root' }
      it { should be_executable.by('owner') }
      it { should be_executable.by('group') }
      it { should be_executable.by('others') }
      it { should be_executable.by_user('root') }
      it { should be_executable.by_user('dd-agent') }
      it { should be_executable.by_user('splunk') }
  end

```

When you have written a set of test cases for your SOEs, store them in git alongside your SOE code or in a dedicated test repository that can be retrived by your EC2 Instances.

## Integrate your tests into a test harness

Now that we have a set of test cases defined for our SOE, we need to integrate them into a test harness.

At it's most basic, A SOE test harness is a set of EC2 Instances that are launched from your AMI (generally in conjunction with a CICD tool) and on bootstrap execute a set of defined tests on the instances to ensure that the functionality and compsition of the AMI is as expected.

Most commonly, this is implemented via a Cloudformation template or a set of terraform modules in which you feed in the AMI ID of your SOE as a parameter and on launch, each EC2 Instance(s) in the harness will 

1. Download and Install the testing tool software (Being careful to not install other dependacies that might result in false positives)
2. Download the latest SOE tests from version control that apply to your SOE release
3. Execute the test suite against your tests with a non-zero exit code representing a test failure on an instance
4. Based on the exit code from step 3 the SOE will be marked as pass or a fail, either

    a. Greenlighting it for release and distribution across the estate

    b. Returning it to the developer for investigation

5. Publish the instances test logs to a central location for later reference and then destroy any un-needed EC2 instances.

## Automate your security scans

It is in this area that I recommend introducing automated security scanning for your instances.

While you have EC2 instances running configuration tests, you can also invoke security scans from tools such as AWS Inspector or Qualys that have scan scheduling and invocation API's alongside running CIS configuration checks at an OS level to ensure your SOEs are compliant to your required security standards.

# Wrapping up

As you can see, some simple changes to how you handle your SOE AMI's development and testing can lead to signfigant results in the reliability and maintainability of your SOEs, especially at scale.

At one customer site alone, we experienced multiple weekly defects resulting in new AMI's being published to end users mid cycle for simple and preventable issues, the introduction of an automated testing framework that must pass prior to distribution resulted in mid cycle SOE releases due to defects going to zero.

I hope this helps you ship better products in your AWS environments.