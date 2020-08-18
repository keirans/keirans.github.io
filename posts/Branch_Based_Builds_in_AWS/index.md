---
layout: post
author: Keiran
title: Branch Based deployment patterns on AWS
---

# Introduction
AWS provides many ways to deploy your applications to its vast ranges of services using an infrastructure as code approach.

There isn't always a "best practice" that suits every single service, application workload, developer worklflow or tooling used, however there are some well oiled approaches that have served myself and my teammates over the years, one of which is the "Branch based build" pattern.

In this blog post, I thought I'd strip this back to basics and walk through the pattern, demonstrating it's objectives, it's capabilities and to it's limitations and considerations for use in your environment as it's something I have taught customers and collegues a like in the past.


# Understanding the branch based build concept

When cloud engineers use an Infrastruture as code approach for resource provisioning, it is best practice to store this code in a version control system such as git to track its history as you would any other code in a system or software product.

An added bonus of doing this, is that when combined with further automation such as a CICD Tool the code in the git repository can be used to automatically deploy the cloud resources required to run your application.

In the Image below, we demonstrate a set of simple steps in which:

1. A CloudFormation template is created that contains:
  * 2 x EC2 Instances in an Autoscale Group
  * An Elastic Load Balancer distributing traffic to the instances
  * A Route53 DNS Record that points to the Load Balancer

2. CloudFormation templates are stored in the master branch of a git repository

3. A Jenkins CICD Job is associated with the master branch of the repository and when changes are pushed to it, the job either creates a new cloudformation stack for the resources, or updates the stack with the changes that have been made to the template in place.


![](img/Basic-CFN-Pipeline.jpg)


# Immutable Infrastructure 


# Branch builds with stateful resources - RDS Snaps


# Thanks and Acknowledgements
Teammates who taught me this over the yars, and also to Gus who's teachings historically I leveraged for this post.