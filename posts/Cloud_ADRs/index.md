---
layout: post
author: Keiran
title: Democratising Cloud development standards with Architecture Decision Records (ADRs)
---

# Introduction

For the better part of the last 18 months, I've been working as an platform engineer on a strategic cloud platform for a large Australian financial institution.

The goal of this initiative was to develop from the ground up a foundational AWS platform suitable for all of the groups cloud workloads from the most critical and regulated  internet facing banking applications to backend HPC clusters. 

This platform is built using a wide range of technologies that includes:

* Amazon Web Services 
* HashiCorp Terraform & Packer 
* Microsoft Powershell
* A variety of programming, configuration management and scripting languages such as
  * Bash
  * Python
  * GoLang
  * NodeJS
  * Puppet

The initial project initially commenced with a core platform engineering team of approximately 10 co-located individuals to build it's [MVP](https://en.wikipedia.org/wiki/Minimum_viable_product) foundations, however it rapidly grew as new features needed to be added and more customers were onboarded onto the platform to use it's capabilities and fix it's defects.

Over the coming months, additional AWS platform engineers were introduced to the team to implement things such as
* Platform wide centralised logging
* Windows and Linux Standard Operating Environments (SOEs)
* Automated Cyber Defence and forensic tooling
* Puppet Masters for long lived instance management
* CICD pipelines for common automation actions
* AWS Account vending
* An extensive library of Terraform modules that would be used by customers to deploy their applications within the platform
* Network deployment and integration

Before we knew it, the 10 person team had reached almost 100, working on a codebase that was hundreds of thousands of lines in size, and as is common in large organisations these individuals were dispersed all over the world, we had representation from Sydney, Melbourne, Auckland, Wellington, Bangalore and more

# Cracks begin to appear

It was around the time that the team had hit the 25 person mark that we observed a drop in developer output and the technical teams started to fall short of their sprint objectives. To make things more interesting this also coincided with the global COVID pandemic which forced all team members to their homes, and the programme of work needing to be delivered remotely. What was once a medium sized team all in a common office was now a fully remote software project.

We spoke to the team members, and dug up some metrics and found that the core of the problems seemed to lie with the way code was being developed and introduced into the environment. Specifically;

* Code quality and development practices were inconsistent and reviews were becoming difficult 
* Code merges and sequencing of releases were challenging 
* Cloud resources and naming standards were not clearly defined or communicated
* Standards were spread out across a variety of knowledge bases and were often communicated via [tribal knowledge](https://en.wikipedia.org/wiki/Tribal_knowledge)
* It became difficult to train and onboard new developers on the platform, or uplift the capabilities of other teams who worked on the platform
* Language selection for things like Lambdas were not clear, resulting in sprawl 
* Tagging standards were not well understood

In addition to the technical challenges, Individuals felt that they did not have a voice or forum to influence the way that the platform's standards were defined and evolved, this lead to;

* Reduced developer output and increased rework on code prior to merging
* Team relationship strain, we were regularly encountering developer frustration at a team and individual level

# Solving this problem with ADR's

We needed a clear way to define and collaborate on our AWS development standards. It was at this stage that one of my customers brought to my attention the concept of "[Architectural Decision Records](https://adr.github.io/)", or ADRs as a lightweight way for us to consolidate our standards. 

## What are ADR's ? 
Familiar to many who are experienced developers, but maybe not to those who have a more traditional infrastructure background such as myself, ADR's are described as

* _An Architectural Decision (AD) is a software design choice that addresses a functional or non-functional requirement that is architecturally significant._
* _An Architectural Decision Record (ADR) captures a single AD, such as often done when writing personal notes or meeting minutes; the collection of ADRs created and maintained in a project constitute its decision log._
  * Source: adr.github.io


## How to they work ? 

Lightweight in nature, each ADR consists of a small markdown document stored in a dedicated git repository that has a predefined set of headings, generally following the below format.


| Heading       | Function      | 
| ------------- | ------------- | 
| Title         | The ID number and title of the ADR being introduced | 
| Status        | The current status of the ADR. Accepted, Rejected, Superseded being the most common | 
| Context       | The technical detail about the ADR, what it's objectives are and the problems it's looking to address, as well as any alternatives that were considered |
| Decision      | The decision that is being made by ratifying the standards proposed in the ADR alongside any comments about discarding any other alternatives | 
| Consequences  | The conseqences of implementing the ADR standards in the environment (both positive and negative) |


_Note: ADR's are focused more on implementation standards, rather than being end to end architecture documents for a particular solution._

As ADRs are stored in git, your version control software such as Github can then be used by your development teams to discuss and revise the proposed standard using the same processes they are familiar with, and when ADR changes are merged to the master branch of the repository they are then considered active.

## How did we implement these ? 

Within our environment, we created the ADR git repository and opted to use the simple [ADR Tools](https://github.com/npryce/adr-tools) to automate the management of the ADR files in conjunction with Github to keep the process consistent.

We then worked with the development team to define a backlog of problematic and missing standards that were causing developer friction of which we started working through to assist the development team, and held a series of team workshops to bring attention and alignment to them as they were phased in.

Out initial ADR's for the AWS platform included: 

| ADR Name      | Objective      | 
| ------------- | ------------- | 
| Terraform data model | Define the approach for keeping business logic out of terraform modules to assist in ongoing maintenance and unit testing | 
| Code Development Standards | Define git commit, branching and pull request standards and setting the expectation of commit messages that contain clear and concise explainations and references to tracking systems such as JIRA tickets   | 
| AWS Resource naming standards | Define the approach to creating AWS resources in a consistent way both at the platform and end user module level | 
| Semantic Versioning for Terraform Modules | Formally communicate that Terraform modules are aligning to the [semver](https://semver.org/) versioning framework  | 


## How did this help ?

As we phased in the ADR process and introduced more standards, we found that developer friction was greatly reduced and and through the clear definition of standards we were able to automate their validation and enforcement in CICD and external governance / compliance systems more effectively increasing developer output and reducing friction.

Reflecting on the project, I wish that I was more aware of the ADR concept and we had implemented them earlier and I'd advocate looking to start any new project of this nature by defining some of these early cloud and development standards even before development commences to pre-empt these occurring and being able to scale your team more effectively earlier.

I encourage you to check out some other ADR examples and reference documentation in the below references which can help you see how other teams implement these for their projects.

* [Architectural Decision Records](https://adr.github.io/)
* [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions.html)
* [Sustainable Architectural Design Decisions](https://www.infoq.com/articles/sustainable-architectural-design-decisions/)
* [Architecture Decision Records at Spotify](https://www.infoq.com/news/2020/04/architecture-decision-records/)
* [The Home Assistant Project - ADRs](https://github.com/home-assistant/architecture/tree/master/adr)
* [ADR tools - ADRs](https://github.com/npryce/adr-tools/tree/master/doc/adr)




