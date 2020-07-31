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

## Use a config management tool
avoid complex shell and bash scripts
Puppet, Chef and others have great Windows support too.




## Declare your user ID's
NFS, EFS, Moving EBS Volumes or Snapshots to new instances.
Try and keep them consistent between Operating systems 
Test for changes

## Use .d directories
avoid configuraiton file editing

* http://blog.siphos.be/2013/05/the-linux-d-approach/
* https://www.redhat.com/sysadmin/etc-configuration-directories


## Rename the Administrator account with caution
EC2Launch and other fun stuff

## Externalise your passwords and runtime params
Don't bake secrets and things into your AMI's that you may need to change later, resulting in unnessasary re-deployment

Build resilience into these callouts, retrys, clear error messages, etc

## Use Powershell to it's potential !

Powershell, DSC, WMF for code consistency across releases
Config management provides hooks into this 

## Don't couple your userdata to a particular tool
- ie puppet for launch directly makes upgrades and changes hard
- Provide a clear interface for launching the SOE from your Terraform modules of CFN Userdata
- 

## Store build time data for later reference and debugging
- Build logs into CWLogs
- Keep the Scan logs from your security tool
- Run an SOS Report during bake, or in your testing tool for diffing
- Need clear tracability of the AMI -> Code Commit or Branch/Tag
- 

## Consider your patch management Approach
Immutable Vs Patch in place

## Publish your AMI ID's through SSM or other means

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
