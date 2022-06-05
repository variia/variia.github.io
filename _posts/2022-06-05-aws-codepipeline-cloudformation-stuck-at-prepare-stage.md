---
title: "AWS Codepipeline Stuck at Cloudformation Prepare Stage"
date: 2022-06-05 11:58:42 +0100
author: Iván Ádám Vári
layout: post
permalink: /aws-codepipeline-stuck-at-cloudformation-prepare-stage/
description: AWS Codepipeline cloudformation prepare stage not completing, appears to be stuck.
keywords: aws,codepipeline,cloudformation,stuck,prepare,stage,incomplete,error,change-set
categories:
  - AWS
tags:
  - aws
  - cloud
  - codepipeline
  - cloudformation
  - cfn
comments: true
sharing: true
footer: true
---
This issue puzzled me and even AWS support for weeks, although for the later it was mostly because of their availability
not their skills:

>I pushed in an update for one of my pipeline managed stacks but the Cloudformation prepare stage was not completing. No
errors, nothing but it was displaying "running" without doing anything really. The Cloudformation change-set
was never created and after a day or two, it eventually timed out.
{: .prompt-info }

As a standard course of action, I logged a support ticket, we went through the usual permission checks, our setup and so
on but there was no clue so support escalated the ticket to backend engineering. After a couple of weeks of hassling our
account manager and support, we got the information we needed and fixed it.

Partially, it was my fault but in the end I think AWS played a key role in this, hence we made a couple of requests to
prevent this to happen in the future: 

* Fix the service to include key backend information in CloudWatch

* Update the documentation 

## Background

We had our Elasticsearch service up for the Opensearch upgrade and it was managed with old, yaml style code and manual
actions. As stated by the <a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-elasticsearch-domain.html#aws-resource-elasticsearch-domain--remarks " target="_blank">documentation</a>,
this change isn't just clicking the **upgrade** button on console. There are specific steps to actually migrate the 
Cloudformation service objects into a new API compatible template and stack.

Since we built a new Codepipeline driven automated deployment framework for CDK generated Cloudformation templates, we
thought how convenient it is, we can move the service to a new template/stack management platform too.

The migration is out of the scope of this article but basically, it included a step where the existing service object
essentially needs to be imported into a new stack then updated and so on. 

And this is where the process failed. It turns out, that Codepipeline does check the existing Cloudformation stack's last
status before proceeding:

>"AWS workers are not able to update the stack because within this operation we treat the key/value pair for
status_status=`IMPORT_COMPLETE` as a state which cannot be updated."
{: .prompt-warning }

This is utterly wrong to me, it is needless to build such a logic into other services, they should just let Cloudformation
to do its job and decide what is valid to update and what is not.

Updating a Cloudformation stack with `IMPORT_COMPLETE` last status is perfectly valid, but is is invalid for Codepipeline
and to be frank, it is nonsense.

## Solution

>Update the stack manually (awscli, console, tool of your choice) with something, edit description, add a parameter,
metadata or whatever you want, all we need is its status changed to `UPDATE_COMPLETE`.
{: .prompt-tip }

Once the stack is out of the `IMPORT_COMPLETE` state, Codepipeline will be be able to update the stack again.
