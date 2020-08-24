---
layout: post
author: Keiran
title: Branch Based deployment patterns on AWS
---

# Introduction
AWS provides many ways to deploy your applications to its vast ranges of services using an infrastructure as code approach.

There isn't always a "best practice" that suits every single service, application workload, developer worklflow or tool, however there are some well oiled approaches that have served my collegues and I well over the years, one of which is the "Branch based build" pattern.

In this blog post, I thought I'd strip this back to basics and walk through the pattern, demonstrating it's objectives, it's capabilities and to some of it's limitations, as it's something I have taught customers and collegues a like in the past.

Note: In this blog post, we will be working with AWS CloudFormation, however, this approach can be implemented with any tool of your choice from the AWS CLI to Hashicorp Terraform.

# From Code to Cloud

When cloud engineers use an Infrastruture as code approach for resource provisioning, it is best practice to store this code in a version control system such as git to track its history as you would for any other system or software product.

An added bonus of doing this, is that when combined with further automation such as a CICD Tool the code in the git repository can be used to automatically deploy the cloud resources required to run your application.

In the Image below, we demonstrate a set of simple steps in which:


1. A CloudFormation template is created that contains:
  * 2 x EC2 Instances in an Autoscale Group
  * An Elastic Load Balancer distributing traffic to the instances
  * A Route53 DNS Record that points to the Load Balancer

2. CloudFormation templates are stored in the master branch of a git repository

3. A Jenkins CICD Job is associated with the master branch of the repository and when changes are pushed or merged to it, it submits the template to the AWS CloudFormation API

4. The CloudFormation service will either create a new CloudFormation stack and then provision the resources, or update an existing stack with the changes that have been made to the template in place.

When changes need to be made to the Cloud resources, the code can be commited to the repository in which the Jenkins job will then trigger a CloudFormation stack update to update and modify your resources.

![](img/Basic-CFN-Pipeline.jpg)


# Challenges and the transition to "Immutable infrastructure"

This deploy and update model lends itself well to core infrastructure such as networks, dns zone config, however when it comes to applications, we often see this approach as something that can introduce problems as rolling back complex application changes, or testing them prior being deployed can be difficult.

As a result, the concept of immutable application deployments arose.

With immutable deployments, the objective is to never upgrade in place deployed application and supporting infrastructure, but rather leverage the flexibility of the cloud providers vast resources, infrastructure as code and CICD automation to provide us the ability to deploy entire copies of subsequent versions of the entire application which use different configuration in which you can transition your users to after testing.

Through leveraging this pattern, in the event a problem is found on new versions, you can easily return the users back to the original copy of the application and it's supporting infrastructure if/when required as it will be running in paralell until you are confident enough that it can be discarded (known as blue/green releases), in addition to this, it also opens the door to new development workflows in which developers can develop and deploy an entire application stack that not just includes the application itself, but the entire supporting infrastructure of virtual machines, operating system configuration, scaling policies, monitoring settings and more.


If you would like to know a little bit more about the concept of immutable infrastructure, I encourage you to check out some of the blog posts and videos below.

* [Hashicorp - What is mutable vs immutable infrastructure ?](https://www.hashicorp.com/resources/what-is-mutable-vs-immutable-infrastructure/)
* [Digital Ocean - What is immutable infrastructure ?](https://www.digitalocean.com/community/tutorials/what-is-immutable-infrastructure)


# Enhancing the development flow
So, How can we enhance the development flow to implement this based on the above example ? 

This is where the the concept of "Branch base builds" comes into play, in this model, we continue to store all of our code in git, however we configure our CICD tool, in this case Jenkins to take a set of actions whenver a new branch is created and pushed to the repository to provision that unique set of defiined resources in that particular branch.

To make things even better, when the branch is deleted, you can also look to implement the destruction of the provisioned resources associated with that branch that are no longer required.

So what do we need to do first ?

## Adding additional parameters to your templates

## 


## Jenkins COnfiguration
- Create a "Multi-Branch Pipeline"
- Link to your reposiotry








![](img/Branch-To-Build-Mapping.JPG)




# Handling stateful resources
# Branch builds with stateful resources - RDS Snaps


# Thanks and Acknowledgements
Teammates who taught me this over the yars, and also to Gus who's teachings historically I leveraged for this post.