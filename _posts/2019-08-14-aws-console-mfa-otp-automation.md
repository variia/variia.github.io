---
title: "AWS MFA enabled console with automated one time password"
date: 2019-08-14 22:18:17 +0100
author: Iván Ádám Vári
layout: post
permalink: /aws-console-mfa-1password-otp-automation/
description: Bash solution, which fetches the OTP from 1Password wallet for MFA enabled AWS console session without user interaction
keywords: osx,macos,otp,1password,aws,mfa,automation,shell,bash,solved
categories:
  - macOS
tags:
  - osx
  - macos
  - 1password
  - aws
comments: true
sharing: true
footer: true
---
Moving to AWS is challenging and fun at the same time. Since our migration progresses well, we have enabled enforced MFA
for IAM accounts, that have Administrator access.

With 1Password OTP, it is simple to setup and easy to use in the WebUI, but I felt there is room for an improvement
for the API calls, `awscli` commands over my console session so I don't have to cut and paste every time my session
token expires.

First thing first, this requires `macOS` (OSX) operating system, `bash`, `awscli`, `jq` utilities and `1Password`.
I assume that, the reader is familiar with the AWS tools and services used here including but not limited to IAM
access, API keys, session tokens and so on.

## AWS config

```bash
$ vim .aws/credentials

[mysession-mfa]
aws_access_key_id = XXXXXXXXXXXXXXXXXXXX
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

[mysession]
aws_access_key_id =
aws_secret_access_key =

```

`mysession-mfa` is the profile name for your IAM access, you should already have an API key and secret configured. Since
our MFA is enforced (the policy used for that is out of the scope of this post), this is very limited. I can only change
my password, set MFA device and basics like that, so we only use it for MFA authentication, nothing else.

`mysession` is the profile name, that will be MFA authenticated and once done, allow me to switch to my Administrator
role and become basically admin user of our account. This has NO key and secret configured, just a blank placeholder.

## System requirements

Ensure you have `bash` as your shell, have `jq` and at last but least have `awscli` installed. The name of your `1Password`
wallet entry, where you have the OTP configured is also crucial. I tend to add the profile name to the title (in brackets)
so it is easy to find: <**AWS IAM (mysession)**>

## The bash function snippet

```bash
$ vim ~/.bashrc

aws_mfa_auth () {
    accountid=${1:-"123456789000"}
    loginid=${2:-"first.last@company.com"}
    profileid=${3:-"mysession"}

    printf "Using account: $accountid\n"
    printf "Using login: $loginid\n"
    printf "Using profile: $profileid\n"

    AWS_MFA_ARN="arn:aws:iam::$accountid:mfa/$loginid"
    AWS_MFA_SESSION_DURATION="28800"

    osascript -e "tell application \"System Events\" to tell process \"1Password mini\"" \
              -e "open location \"onepassword://extension/search/$profileid\"" \
              -e "end tell" \
              -e "delay 1" \
              -e "tell application \"System Events\" to tell process \"1Password mini\"" \
              -e "keystroke \"c\" using {command down, shift down, control down}" \
              -e "end tell" \
              -e "delay 1"

    authcode=$(pbpaste)

    output=$(aws --profile ${profileid}-mfa sts get-session-token --serial-number ${AWS_MFA_ARN} --duration-seconds ${AWS_MFA_SESSION_DURATION} --token-code $authcode)

    AWS_ACCESS_KEY_ID=$(jq -r '.Credentials.AccessKeyId' <<< $output)
    AWS_SECRET_ACCESS_KEY=$(jq -r '.Credentials.SecretAccessKey' <<< $output)
    AWS_SESSION_TOKEN=$(jq -r '.Credentials.SessionToken' <<< $output)

    aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID} --profile $profileid
    aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY} --profile $profileid
    aws configure set aws_session_token ${AWS_SESSION_TOKEN} --profile $profileid

    (sleep ${AWS_MFA_SESSION_DURATION} && osascript -e "display notification \"MFA token expired\" with title \"AWS $profileid\"" &)
    exit 0
  }
```

As shown, the  `account`, `login` and `session` IDs have defaults but overridable. I tend to set the session duration to
8+ hours so I don't have to authenticate multiple times a day especially when we are in heavy development.

`osascript` takes care of fetching the OTP password from the correct wallet entry and at completion, `awscli` updates
the placeholder entry with the temp API key and secret.

The last part is bonus: a basic desktop notification that the session token expired.

Finally, just watch it doing its thing...

```bash
$ aws_mfa_auth
```

Once the shell window closed, you could open another and try anything that you have permissions for BUT remember to use
your MFA session profile like this:

```bash
$ aws ec2 describe-instances --region eu-central-1 --profile mysession --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],State.Name,PrivateIpAddress,PublicIpAddress]' --output text | sort -k2 | column -t
```

This would list all running EC2 instances in a nice, column sorted window.


>Make sure **your wallet is unlocked** before running the function. This solution does not wait long for the wallet to be
unlocked so if you are slow, it bombs out after few seconds.
{: .prompt-tip }
