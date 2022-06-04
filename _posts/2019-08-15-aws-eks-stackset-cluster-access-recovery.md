---
title: "AWS EKS StackSet created cluster access recovery"
date: 2019-08-16 13:23:34 +0100
author: Iván Ádám Vári
layout: post
permalink: /aws-eks-stackset-cluster-access-recovery/
description: Recover EKS access for clusters that are created by StackSet
keywords: aws,eks,stackset,cfn,cloudformation,access,recovery,solved
categories:
  - AWS
tags:
  - aws
  - cloud
  - eks
  - cloudformation
  - cfn
  - stackset
comments: true
sharing: true
footer: true
---
Kubernetes is contagious and nowadays hard to ignore. So we decided to look into EKS to see how it would work for
our microservice suite. As of today, there are 2 ways of creating (official) an EKS cluster: `eksctl` via CLI or
point and click through the Web-UI.

We have multiple accounts and use services in multiple regions, so I developed a custom CloudFormation template to
build our EKS cluster with StackSets. While this worked perfectly, I found an issue of not being able to access the
cluster at all.

<a href="https://docs.aws.amazon.com/eks/latest/userguide/troubleshooting.html#unauthorized" target="_blank">It turned out to be perfectly normal</a>,
since when an Amazon EKS cluster is created, the IAM entity (user or role) that creates the cluster is added to the
Kubernetes RBAC authorization table as the administrator _(with system:master permissions)_. Initially, only that IAM
user (initiated the cluster creation) can make calls to the Kubernetes API server using `kubectl`.

## About StackSets

<a href="https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs.html" target="_blank">The basics of StackSets,</a>
is that you create a role **AWSCloudFormationStackSetAdministrationRole** on your **MASTER** account (where the StackSet
is launched from) and a role **AWSCloudFormationStackSetExecutionRole** on your **TARGET** account(s) where StackSets
will manage CloudFormation stacks for you.

## Assume role setup

So in this case, you need to make sure that your IAM user entity can assume (temporarily) the **AWSCloudFormationStackSetExecutionRole**.
This can be done multiple ways, but the easiest would be to create a TEMP group, attach an inline policy to it and at
completion, make your IAM user member of this TEMP group.

The inline policy for your TEMP group:

```json
{
        "Version": "2012-10-17",
        "Statement": [
          {
            "Action": [
              "sts:AssumeRole"
            ],
            "Resource": [
              "arn:aws:iam::123456789100:role/AWSCloudFormationStackSetExecutionRole",
            ],
            "Effect": "Allow"
          }
        ]
}
```

The `<123456789100>` number is YOUR target account ID where the EKS cluster is created.

Create a TEMP AWS config entry:

```bash
$ vim ~/.aws/config

[profile eks]
region = eu-central-1
role_arn = arn:aws:iam::123456789100:role/AWSCloudFormationStackSetExecutionRole
source_profile = <your_normal_iam_profile_name>
``` 

Test your assume role before proceeding:

```bash
$ aws --profile eks sts assume-role --role-arn arn:aws:iam::123456789100:role/AWSCloudFormationStackSetExecutionRole --role-session-name test
```

## Update Kubernetes RBAC

<a href="https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html" target="_blank">Get the ConfigMap template</a> from AWS:

```bash
$ curl -o aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/aws-auth-cm.yaml
```

Edit it:

{% raw %}
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::123456789100:role/StackSet-EKS-c4c34dcd-45as-11-NodeInstanceRole-34F45DD54RSKK
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
    - rolearn: arn:aws:iam::123456789100:role/Administrator
      username: admin
      groups:
        - system:masters 
```
{% endraw %}

The first element of the `mapRoles` is required for the worker nodes, so they can join the cluster automatically.
For this, you need to get the ARN of the InstanceRole that is attached to your nodes, normally it is generated 
on the fly when the stack is created.

The second element is our required admin access details. In my case a special role `Administrator`, that only certain
users can assume, but <a href="https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html"
target="_blank">you could also add IAM user arn.</a>

When ready, get a copy of your cluster config with your new TEMP profile, then update the cluster:

```bash
$ aws --profile eks --region eu-central-1 eks update-kubeconfig --name <eks cluster name you created>
$ kubectl apply -f aws-auth-cm.yaml
```

At completion, delete the kube config you got with your TEMP profile, get a new one with your normal
aws profile. Alternatively, edit the config and update the AWS_PROFILE key at the bottom...

```
$ rm ~/.kube/config
$ aws --profile <normal aws profile name> --region eu-central-1 eks update-kubeconfig --name <eks cluster name you created>
``` 

From this point on, your normal IAM user profile should be able to access the cluster:

```bash
$ kubectl get svc
$ kubectl describe configmap -n kube-system aws-auth
$ kubectl get namespace
$ kubectl get nodes
```

>Do not forget to remove the TEMP group and the TEMP AWS config profile at completion.
{: .prompt-tip }
