---
layout: single
---
# Docker everyday on OSX

Disclaimer: This breaks a bunch of the core Docker principals for application deployment, and this works for my workflow. This should actually work on all OS's that can run Linux docker images such as Windows and Linux itself, however YMMV 

## Introduction
I use a mac laptop every day for my technical and personal work. It's old, but it has quite a bit of power for the things I need to do every day.


```
#
# Create a docker image suitable for day to day use when on client
# sites rather than pollute the OSX Base OS on my laptop.
# Yes this is pretty heavy for a container....
#
FROM centos:7
MAINTAINER Keiran Sweet "Keiran@gmail.com"
RUN yum -y update
RUN yum -y install epel-release
RUN yum -y install which git vim mlocate curl sudo unzip file python-devel python-pip python34 python34-devel wget bind-utils
```
