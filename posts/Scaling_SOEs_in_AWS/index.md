---
layout: post
author: Keiran
title: Scaling Standard Operating Environments in AWS 
---

# Introduction to SOE's

A Standard Operating Environment (SOE) implements and standardizes an operating system and its associated software, resulting in the reduction in the cost and time taken to deploy, configure, maintain, support, secure and manage the operating systems across a large fleet of virtual machine instances.

In AWS environments, SOE's are distributed in the for of Amazon Machine Images (AMIs), it is not uncommon for environments of all sizes to have multiple SOE flavors that cover different Operating System types and releases, such as Microsoft Windows and various distributions of Linux and the subversions of each.

In environments that follow the immutable infrastructure model, the SOE is released on monthly or similar cadence providing new features and capabilities, security patches and more, with mid-cycle releases often occuring to cater for emergency security patches and fixes.

Once updated SOE's are produced, it then is up to downstream application teams to adopt the latest SOE as part of their continuous delivery practices and re-deploy their applications ontop of it.

In the many AWS environments i've worked in over the years, I have observed organisations that have in excess of 10+ unique SOE's that need to be distributed to tens, of not hundreds of AWS accounts for use by a wide variety of application teams.

In this post, I'll detail some of the learnings I have had over the years to ensure that when building SOE's in AWS you produce a high quality product for consumption by your users.

# Pre-Development steps to success
## Gather clear requirements and building solid process

Before jumping into the fun stuff of writing code and building AMI's for use, the most important thing that you need to think about is the requirements you are working to and the processes you are going to implement and follow to ensure that you are always producing a high quality product for all users.

When speaking to clients about SOE's, I always emphasise the need to treat each SOE as a software product that needs to align to common development standards, including:

- What is this image going to be used for ? 
  - Will it be used Immutable Vs Long Lived Instances and what is the required release cadence for each
  - What are the types of applications it will be hosting and what packages and configuration settings will it need to have to support them
  - How will the end user consume this SOE (ie, Terraform Modules, Cloudformation Templates, Service Catalogue)

- How will end users discover and use this AMI ? 
  - Where will the user guide be to make it clear and easy to use ? 
  - How will end users and their applications programatically discover the latest AMI's from their automation pipelines
  - How will the release notes be distributed with each release so that changes are well understood ?
  - How will end users and maintainers track enhancement and defects to ensure they are addressed and delivered clearly ?

In the case of release notes and issue tracking, it is my opinion that all SOE related tasks are done so in JIRA (or a similar tracking tool), and release notes are automatically generated, reducing manual work.

## Buy or Build - Evaluating the impact of using marketplace AMI's
Many organisations often opt to use pre-configured AMI's from the AWS Marketplace to get up and running faster, using these AMI's as the basis for their SOE's instead of the Vanilla ones from Microsoft, Redhat or Ubuntu.

Although these can help, you should consider the following potential implications before use to ensure that they don't come with unforseen consequences.

Some things to think about are;

* Many marketplace AMI's have additional costs per hour. Initially a few cents in the dollar per hour may not seem like much, however when you are running thousands of EC2 instances this can add up to thousands of dollars very quickly.

* Not all marketplace AMI's support sub-hour billing granularity. If you have a workload that may benefit for per second billing, make sure you aren't basing your SOE off an image that doesnt support it.

* Who is going to Support the AMI if you find issues with it in the future, and what is their SLA for defect resolution

* What are the implications to your environment if a marketplace is AMI is deprecicated or pulled from the marketplace suddenly ? I've encountered SOE's not being available in newly created AWS accounts because the Source AMI it is built from has been depreciated from the vendor.


## Start small and increment

As the [AWS Well Architected framework emphasises](https://d1.awsstatic.com/whitepapers/architecture/AWS_Well-Architected_Framework.pdf), when building your SOEs out look to make "Make frequent, small, reversible changes".

It is a much more effective approach to building AMI's to take a base image and implement the features and capabilities in a gradual and phased approach to ensure that each peice of code is linked to a particular feature or configuration request.


One common problem I have regularly observed is teams taking large hardening and configuration scripts from on-premise systems and applying them to their AWS SOEs hoping that they will provide similar security or configuration outcomes. In most cases, this results in the AMI's become polluted with settings that are not applicable to EC2 instances, or breaking functionality required by services such as EC2Launch.

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
After building and working on AWS SOE's for quite some time, here are some tips from the trenches to help you build better AMI's for your organisations.


## Don't disable SELinux
* [It's 2020](https://stopdisablingselinux.com/), and rolling it out to your fleet after you have turned it off is not a very fun exersize.

## Use a Configuration management tool
Configuration management tools shine by helping you configure virtual machines in a delcarative way, resulting in a reduction in large complex bash and powershell scripts.

In addition to this, many modern configuration management tools such as Puppet, Chef and Ansible provide you with additional functionality such as
* Easily handling differences between Operating System types and versions for common tasks like
  * Software and Package Management (apt, rpm, yum, deb, msi, , gem, pip, exe)
  * User and Group Management
  * File distribution and permission management
* Creating configuration files from templates
* Providing a library of modules and example code to get you moving faster
* Out of the box integration with Image baking tools such as Packer

A common misconception with is that Windows does not lend itself well to configuration management tools, however tools such as Chef and Puppet having mature Windows support with the [later having an extensive library of Windows modules for use.](https://forge.puppet.com/puppetlabs/windows)


## Declare your Linux user / group ID's and keep them consistent
A common problem that harks back to the days of more traditional system administration practices when Linux User/Group ID numbers change between SOE releases or virtual machine instances.

This becomes a signifigant problem with solutions that use shared filesystems such as EFS or leverage EBS snapshots, as changes in the User ID number can result in files and directories being owned by different users than required.

The best solution to this is to ensure that all software that is installed that creates user accounts have an ID registered for them in a central database, and then in the SOE code explicitly create these users / groups does so with their assigned ID. Where possible, keep these UID's consistent across all your Linux distributions.

For identities for humans, it's generally best to not place these identities in the SOE, but to integrate the instance with an external directory service such as LDAP or Active Directory.


## Use Linux .d directories 
The Linux operating system has hundreds of configuration files, many of which need settings changed to support the requirements of the application teams and business units.

Updating configuration files in place is not often easy as their formats can vary, and although there are tools such as [Augeas](https://augeas.net/) to help with doing this in an idempotent way, configuration file changes often trend back to unmanageable sed commands run over configuration files in loops.

Fortunately, many services configuration files in Linux provide a .d directory framework that lets you drop off complimentary configuration files which are then evaluated in conjunction with the main configuration file, resulting in there being no need to edit large complex configuration files in place.

This approach to configuring services also lends itself well to configuration management tools which excel at templating files and distributing small files.

Some good articles about this functionality of the Linux OS can be found below

* [The Linux .d Approach](http://blog.siphos.be/2013/05/the-linux-d-approach/)
* [etc configuration directories](https://www.redhat.com/sysadmin/etc-configuration-directories)


## Rename the Administrator account with caution
Many Security hardening guides advocate the renaming of the Administrator account to something else to lessen the risk of brute force attacks.

Prior to the release of EC2Launch v2 , this broke EC2Launch preventing the "Get Windows Password" functionality in the AWS console, as well as tools such as Packer that used the corrosponding API call.

As a result, I personally recommend avoiding this process, protecting the Administrator account with a strong password and other measures.

## Externalise your passwords and runtime params
When a SOE AMI is used to launch a EC2 Instance a number of actions are invoked through the instance user data or launch scripts baked into the AMI itself.

These actions often include tasks such as:
* Starting services suchs as monitoring tools and security agents
* Joining the instance to Active Directory for user authentication and group policy management
* Integrating the instances into downstream services such as configuration management tools or software repositories

Many of these steps require the instance to have access to passwords or run time configuration information to suceed. When building SOE AMI's it is best to not hardcode these values into the image itself, but develop bootstrap logic that sources this from an external source such as AWS SSM Parameter Store or AWS Secrets manager.

Taking this approach will enasure that the SOE is free from credentials as well as portable across environment without the need for producing additional image updates.

When externalising these runtime values, it's important to add resilience into the scripts that fetch them via to retry anbd backoff logic.


## Use Powershell to it's potential across all your Windows Versions

Powershell is an incredibly powerful tool for Windows Administrators.  Unfortunately depending on the version of Powershell and the features it provides vary between Windows releases, often resulting in Powershell scripts being written that leverage the lowest common denominator feature or capability that is available.

But not all is lost !

Microsoft provides the "Windows Management Framework" 

WMF installation adds and/or updates the following features:

* Windows PowerShell
* Windows PowerShell Desired State Configuration (DSC)
* Windows PowerShell Integrated Script Environment (ISE)
* Windows Remote Management (WinRM)
* Windows Management Instrumentation (WMI)
* Windows PowerShell Web Services (Management OData IIS Extension)
* Software Inventory Logging (SIL)
* Server Manager CIM Provider

## Leave your timezone at UTC 

* https://twitter.com/QuinnyPig/status/997178792257830912

## Store build time data for later reference and debugging
- Build logs into CWLogs
- Keep the Scan logs from your security tool
- Run an SOS Report during bake, or in your testing tool for diffing
- Need clear tracability of the AMI -> Code Commit or Branch/Tag
- 

# Automated Testing and Debugging

## Build a test harness early
Test during the bake phase

Test During the Launch Phase

Go deep on functionality testing

Test across instance types

Can you rebake it ? Or have you introduced a defect.

User Pester

## Automate your security scans
Inspector, Qualys, etc all have API's
Launch instances and automatically scan them prior to releasing the SOE to customers.

## Build a debug mode into your SOEs and Launches
