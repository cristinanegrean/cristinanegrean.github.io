---
layout: post
author: Cristina Negrean
title: "The Developer's Lift and Shift Guide to Elastic Kubernetes Service"
image: /img/gateway_eks_k9s.png
tags: [Bash, AWS IAM, AWS STS, AWS Vault, AWS EKS, K9s, AWS SSM, Kubernetes, K8s]
category: Cloud Native Development
---
## The developer dilemma

I've been getting a lot the following developer questions on my current assignment: 
* "Hi, I have most of my box back up and running. I can use k9s for the environments, but it requires me to do some aws-vault commands to switch between clusters. Do you have a script that allows met to switch more easily? Does the script play nice with the aws plugin in IntelliJ IDEA? For now, I've just removed the plugin, back to the command line."
* Can you change for me application service X configuration Y to new value Z on environment W? 

Granted that AWS and Kubernetes introduce complexity for the average Java developer, I thought it is time for a new blog post.
The post also assumes the developer has [a single IAM user](https://aws.amazon.com/blogs/security/how-to-use-a-single-iam-user-to-easily-access-all-your-accounts-by-using-the-aws-cli/) 
and that operations has set up an IAM role to [delegate access across AWS accounts](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html).

The post is written with a focus on macOS, but it should work on Linux devices as well, although no guarantees. It should be fairly easy to transport the concepts to Windows devices.

## Prerequisites

* Command line shell (bash or zsh)
* [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
* [k9s](https://k9scli.io/topics/install/)
* [aws-vault](https://github.com/99designs/aws-vault)
* [kubectl](https://kubernetes.io/docs/tasks/tools/)

On macOS using Homebrew package manager, dependencies installation is a one liner: 

`brew install awscli kubectl aws-vault derailed/k9s/k9s`

## Access to AWS using aws-vault

So you've installed and set up all the prerequisites and your operation guy or girl provided you
with your AWS IAM username and credentials. Next things to do are:

1) Set up your AWS Multi-factor Authentication device

AWS Multi-Factor Authentication (MFA) is a simple best practice that adds an extra layer of protection on top of your 
basic AWS credentials. You can choose between a virtual device, like Google Authenticator app on your mobile, or a physical one
like a YubiKey. With the virtual device you will need to provide the MFA token later. 

2) Store your AWS credentials in AWS Vault

`aws-vault` supports several backends to store your credentials. My preference is to use an encrypted file. 
So, you need to add the following variable to your `~/.zshrc` or `~/.bash_profile`: `export AWS_VAULT_BACKEND="file"`, 
after which run `aws-vault add <user>`. Example if your IAM user is `cristina`:

```bash
➜  ~ aws-vault add cristina
Enter Access Key ID: mysecretaccesskeyid
Enter Secret Access Key: 
Added credentials to profile "cristina" in vault
➜  ~ 
```

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> On macOS you can actually import the aws-vault keychain into Keychain Access app, and edit it to extend the `Lock after X minutes of inactivity` time, in case you don't want to type AWS Vault file passphrase all the time.

3) Create named profiles

The AWS CLI supports using any of multiple named profiles that are stored in the config and credentials files.
Once you've installed `AWS CLI` you will notice a `~/.aws/` (Linux & Mac) or `%USERPROFILE%\.aws\` (Windows) directory.
As we're using `aws-vault` for aws credentials storage, make sure to delete any previously configured `credentials` file.
Also check your `.bash_profile` or `.zshrc` that you're not exporting any environment variables for configuring AWS.

My directory looks like below:

```bash
➜  ~ ls .aws
config
➜  ~ 
```

Find out which AWS accounts you have developer access to and setup named profiles for each of them.

An example of `~/.aws/config` file for developer access to `dev`, `tst` and `acc` AWS accounts:

```
[default]
region=eu-central-1

[profile <your user name>]
region=eu-central-1
mfa_serial=arn:aws:iam::<iam aws account>:mfa/<your user name>

[profile dev]
region=eu-central-1
source_profile=<your user name>
role_arn=arn:aws:iam::<dev aws account>:role/OrganizationAccountAccessRole
mfa_serial=arn:aws:iam::<iam aws account>:mfa/<your user name>

[profile tst]
region=eu-west-1
source_profile=<your user name>
role_arn=arn:aws:iam::<tst aws account>:role/OrganizationAccountAccessRole
mfa_serial=arn:aws:iam::<iam aws account>:mfa/<your user name>

[profile acc]
region=eu-central-1
source_profile=<your user name>
role_arn=arn:aws:iam::<acc aws account>:role/OrganizationAccountAccessRole
mfa_serial=arn:aws:iam::<iam aws account>:mfa/<your user name>
```

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> If the development source code repo contains as well some Terraform code, you may find the AWS account numbers configured in there.

## AWS EKS Access

Now that we have set up AWS vault and the named profile, we can write a `bash script` based on `aws-vault` commands to switch between Elastic Kubernetes Service clusters the operation guy or girl has set up developer access for.

```bash
#!/usr/bin/env bash

PROFILE=$1
[ -z "$PROFILE" ] && echo "You need to provide a profile name" && exit 1
REGION=${2:-eu-central-1}
aws-vault --debug exec ${PROFILE} -- aws eks update-kubeconfig --name nlo-${PROFILE}-gateway --region ${REGION}
aws-vault --debug exec ${PROFILE} -- k9s
#
```
Listing 1: `cat auth-eks.sh`

Do not forget to make it executable: `chmod +x auth-eks.sh`

Let's depict the Bash script and how it works:
* validates and documents its permitted usage patterns `./auth-eks.sh <profile>` or `./auth-eks.sh <profile> <region>`
* in case no region command line argument is given, REGION bash script variable will default to value `eu-central-1`. The default is based on the fact that all EKS clusters I have developer access to, in my current assignment, are provisioned in AWS region Europe (Frankfurt), except for the `tst` AWS account and profile, where that is `eu-west-1` aka Europe (Ireland). Hence the region is an optional command line argument, thus adapt this to your own use case.
* returns a set of temporary security credentials in a [session](https://github.com/99designs/aws-vault/blob/master/USAGE.md#managing-sessions) that you can use to access Amazon Web Services resources (Elastic Kubernetes Service is one such resource) in the `<profile>` AWS account, where profile command line argument should be one of the profiles you've defined in `~/.aws/config`. Note you most likely will not have access to any resources in the AWS account where your IAM user resides, thus the first command line argument will be: `dev`, `tst` or `acc`
* for in-depth understanding, note that `aws-vault exec` uses in the background the `AWS Security Token Service API` to generate the set of temporary credentials via the `GetSessionToken` or `AssumeRole` API calls:
```bash
aws sts assume-role \
  --role-arn "$ROLE_ARN" \
  --role-session-name "session-${PROFILE}-$(date +%s)" \
  --profile "$IDENTITY_PROFILE" \
  --serial-number "$MFA_SERIAL_NUMBER" \
  --duration-seconds 3600 \
  --token-code "$TOKEN"
```
* the set of temporarily credentials within the `aws-vault session` is then used to switch context to the EKS cluster named `nlo-${PROFILE}-gateway` in either command line argument or default set by script AWS region. Note that having a consistent naming convention of Kubernetes clusters hosting same application, hereby `gateway`, services across AWS accounts pertinent to `dev`, `tst`, `acc` is desired. Replace `gateway` with your own naming.
* the switching of context is necessary as we assume you have [access to multiple clusters](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) via your `kubeconfig file`: `~/.kube/config`
* last bit of the script connects to the Kubernetes cluster to which you've switch context to in previous `asw-vault` command

Note how the script uses  `--debug` so that you can see directly in the command line how it works.

```
2021/11/12 15:31:43 aws-vault v6.3.1
2021/11/12 15:31:43 Loading config file /Users/cristinanegrean/.aws/config
2021/11/12 15:31:43 Parsing config file /Users/cristinanegrean/.aws/config
2021/11/12 15:31:43 [keyring] Considering backends: [file]
2021/11/12 15:31:43 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:43 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:43 profile cnegrean: using stored credentials
2021/11/12 15:31:43 profile cnegrean: using GetSessionToken (with MFA)
2021/11/12 15:31:43 profile dev: using AssumeRole (chained MFA)
2021/11/12 15:31:43 Using STS endpoint https://sts.amazonaws.com
2021/11/12 15:31:43 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:43 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:43 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:43 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:43 Re-using cached credentials ****************G6JQ from sts.GetSessionToken, expires in 1h12m41.177877s
2021/11/12 15:31:44 Generated credentials ****************QHER using AssumeRole, expires in 59m59.592323s
2021/11/12 15:31:44 Setting subprocess env: AWS_DEFAULT_REGION=eu-central-1, AWS_REGION=eu-central-1
2021/11/12 15:31:44 Setting subprocess env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
2021/11/12 15:31:44 Setting subprocess env: AWS_SESSION_TOKEN, AWS_SECURITY_TOKEN
2021/11/12 15:31:44 Setting subprocess env: AWS_SESSION_EXPIRATION
2021/11/12 15:31:44 Exec command aws eks update-kubeconfig --name nlo-dev-gateway --region eu-central-1
2021/11/12 15:31:44 Found executable /usr/local/bin/aws
Updated context arn:aws:eks:eu-central-1:10xxxxxxx3:cluster/nlo-dev-gateway in /Users/cristinanegrean/.kube/config
2021/11/12 15:31:45 aws-vault v6.3.1
2021/11/12 15:31:45 Loading config file /Users/cristinanegrean/.aws/config
2021/11/12 15:31:45 Parsing config file /Users/cristinanegrean/.aws/config
2021/11/12 15:31:45 [keyring] Considering backends: [file]
2021/11/12 15:31:45 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:45 profile cnegrean: using stored credentials
2021/11/12 15:31:45 profile cnegrean: using GetSessionToken (with MFA)
2021/11/12 15:31:45 profile dev: using AssumeRole (chained MFA)
2021/11/12 15:31:45 Using STS endpoint https://sts.amazonaws.com
2021/11/12 15:31:45 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 15:31:45 Re-using cached credentials ****************G6JQ from sts.GetSessionToken, expires in 1h12m39.128717s
2021/11/12 15:31:46 Generated credentials ****************O6DI using AssumeRole, expires in 59m59.542313s
2021/11/12 15:31:46 Setting subprocess env: AWS_DEFAULT_REGION=eu-central-1, AWS_REGION=eu-central-1
2021/11/12 15:31:46 Setting subprocess env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
2021/11/12 15:31:46 Setting subprocess env: AWS_SESSION_TOKEN, AWS_SECURITY_TOKEN
2021/11/12 15:31:46 Setting subprocess env: AWS_SESSION_EXPIRATION
2021/11/12 15:31:46 Exec command k9s 
2021/11/12 15:31:46 Found executable /usr/local/bin/k9s                                                                          
```
Listing 2: `./auth-eks.sh dev`

## K9s — the powerful terminal UI for Kubernetes

Note how in the prerequisites, I mention installing `kubectl` which is the de-facto CLI for interacting with Kubernetes clusters?
If you have experience with it, great, use it, however your average Java developer will have a steep learning curve into `Kubernetes (K8s)` and interacting with `K8s Cluster Objects` via `kubectl`.
Here is where [K9S](https://github.com/derailed/k9s) comes to rescue, providing a terminal-based UI to manage Kubernetes clusters that aims to simplify navigating, observing, and managing your applications running in `K8S` or its public cloud hosted variants: `EKS` and `GKE`.
The application uses the standard `.kube/config` file, the same way `kubectl` is working.

Below is how it looks in action, after I've obfuscated some IP details:
<img class="img-responsive" src="{{ site.baseurl }}/img/posts/developers-free-lift-to-eks/gateway_eks_k9s.png"/>


## Basic Navigation

Here are the commands for most common Kubernetes resources:
* `no` — Nodes;
* `ns` — Namespaces;
* `deploy` — Deployments;
* `po` — Pods;
* `svc` — Services;
* `ing` — Ingresses;
* `rs` — ReplicaSets;
* `cm` — ConfigMaps;

## Changing application service X configuration Y to new value Z on environment W

In my assignment, we're using [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
to centralize application services configuration. Configuration values are read at application service start-up. The application services also heavily rely on feature flags to guide business logic and API integrations.
Such a feature flag is corresponding to an entry in AWS Systems Manager Parameter Store. As requests to temporarily change feature flag value happen all the time, I've simplified my life by creating yet another custom `Bash script`:

```bash
#!/usr/bin/env bash

PROFILE=$1
CONFIG=$2
VAL=$3
DEPLOYMENT=$4
NAMESPACE=$5
[ -z "$PROFILE" -o -z "$CONFIG" -o -z "$VAL" -o -z "$DEPLOYMENT" -o -z "$NAMESPACE" ] && echo "You need to provide a profile name, ssm config name, ssm config value to set, kubernetes deployment name to restart, kubernetes namespace deployment resides" && exit 1
REGION=${6:-eu-central-1}
echo "PROFILE=${PROFILE} ; CONFIG=${CONFIG} ; VAL=${VAL}; DEPLOYMENT=${DEPLOYMENT}; NAMESPACE=${NAMESPACE}; REGION=${REGION}" >&2
aws-vault --debug exec ${PROFILE} --region ${REGION} -- aws ssm put-parameter --name /config-${PROFILE}/application/${CONFIG} --type String --value "${VAL}" --overwrite >&2
aws-vault --debug exec ${PROFILE} --region ${REGION} -- aws ssm put-parameter --name /config-${PROFILE}/${DEPLOYMENT}/${CONFIG} --type String --value "${VAL}" --overwrite >&2
aws-vault --debug exec ${PROFILE} --region ${REGION} -- kubectl rollout restart deployment.apps/${DEPLOYMENT} -n ${NAMESPACE} >&2
#
```
Listing 3: `cat ssm-config-overwrite.sh`

How it works:

```
➜  ~ ./ssm-config-overwrite.sh tst test.config someRandomValue game-process gateway-private eu-west-1  
PROFILE=tst ; CONFIG=test.config ; VAL=someRandomValue; DEPLOYMENT=game-process; NAMESPACE=gateway-private; REGION=eu-west-1
2021/11/12 17:00:11 aws-vault v6.3.1
2021/11/12 17:00:11 Loading config file /Users/cristinanegrean/.aws/config
2021/11/12 17:00:11 Parsing config file /Users/cristinanegrean/.aws/config
2021/11/12 17:00:11 [keyring] Considering backends: [file]
2021/11/12 17:00:11 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 17:00:11 profile cnegrean: using stored credentials
2021/11/12 17:00:11 profile cnegrean: using GetSessionToken (with MFA)
2021/11/12 17:00:11 profile tst: using AssumeRole (chained MFA)
2021/11/12 17:00:11 Using STS endpoint https://sts.amazonaws.com
2021/11/12 17:00:11 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 17:00:11 Re-using cached credentials ****************6T2I from sts.GetSessionToken, expires in 7h58m44.205944s
2021/11/12 17:00:12 Generated credentials ****************ZMBN using AssumeRole, expires in 59m59.520134s
2021/11/12 17:00:12 Setting subprocess env: AWS_DEFAULT_REGION=eu-west-1, AWS_REGION=eu-west-1
2021/11/12 17:00:12 Setting subprocess env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
2021/11/12 17:00:12 Setting subprocess env: AWS_SESSION_TOKEN, AWS_SECURITY_TOKEN
2021/11/12 17:00:12 Setting subprocess env: AWS_SESSION_EXPIRATION
2021/11/12 17:00:12 Exec command aws ssm put-parameter --name /config-tst/application/test.config --type String --value someRandomValue --overwrite
2021/11/12 17:00:12 Found executable /usr/local/bin/aws
2021/11/12 17:00:19 aws-vault v6.3.1
2021/11/12 17:00:19 Loading config file /Users/cristinanegrean/.aws/config
2021/11/12 17:00:19 Parsing config file /Users/cristinanegrean/.aws/config
2021/11/12 17:00:19 [keyring] Considering backends: [file]
2021/11/12 17:00:19 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 17:00:19 profile cnegrean: using stored credentials
2021/11/12 17:00:19 profile cnegrean: using GetSessionToken (with MFA)
2021/11/12 17:00:19 profile tst: using AssumeRole (chained MFA)
2021/11/12 17:00:19 Using STS endpoint https://sts.amazonaws.com
2021/11/12 17:00:19 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 17:00:19 Re-using cached credentials ****************6T2I from sts.GetSessionToken, expires in 7h58m36.607079s
2021/11/12 17:00:19 Generated credentials ****************3U23 using AssumeRole, expires in 59m59.025258s
2021/11/12 17:00:19 Setting subprocess env: AWS_DEFAULT_REGION=eu-west-1, AWS_REGION=eu-west-1
2021/11/12 17:00:19 Setting subprocess env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
2021/11/12 17:00:19 Setting subprocess env: AWS_SESSION_TOKEN, AWS_SECURITY_TOKEN
2021/11/12 17:00:19 Setting subprocess env: AWS_SESSION_EXPIRATION
2021/11/12 17:00:19 Exec command aws ssm put-parameter --name /config-tst/game-process/test.config --type String --value someRandomValue --overwrite
2021/11/12 17:00:19 Found executable /usr/local/bin/aws
2021/11/12 17:00:23 aws-vault v6.3.1
2021/11/12 17:00:23 Loading config file /Users/cristinanegrean/.aws/config
2021/11/12 17:00:23 Parsing config file /Users/cristinanegrean/.aws/config
2021/11/12 17:00:23 [keyring] Considering backends: [file]
2021/11/12 17:00:23 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 17:00:23 profile cnegrean: using stored credentials
2021/11/12 17:00:23 profile cnegrean: using GetSessionToken (with MFA)
2021/11/12 17:00:23 profile tst: using AssumeRole (chained MFA)
2021/11/12 17:00:23 Using STS endpoint https://sts.amazonaws.com
2021/11/12 17:00:23 [keyring] Expanded file dir to /Users/cristinanegrean/.awsvault/keys/
2021/11/12 17:00:23 Re-using cached credentials ****************6T2I from sts.GetSessionToken, expires in 7h58m32.852906s
2021/11/12 17:00:23 Generated credentials ****************YFGD using AssumeRole, expires in 59m59.213069s
2021/11/12 17:00:23 Setting subprocess env: AWS_DEFAULT_REGION=eu-west-1, AWS_REGION=eu-west-1
2021/11/12 17:00:23 Setting subprocess env: AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY
2021/11/12 17:00:23 Setting subprocess env: AWS_SESSION_TOKEN, AWS_SECURITY_TOKEN
2021/11/12 17:00:23 Setting subprocess env: AWS_SESSION_EXPIRATION
2021/11/12 17:00:23 Exec command kubectl rollout restart deployment.apps/game-process -n gateway-private
2021/11/12 17:00:23 Found executable /usr/local/bin/kubectl
deployment.apps/game-process restarted
➜  ~ 
```
Listing 4: `./ssm-config-overwrite.sh tst test.config someRandomValue game-process gateway-private eu-west-1`

## Summary

Hopefully my brain dump will inspire a new generation of Java developers, at least on my assignment, the `cloud native` one `:smile:` and I will get the chance to get some feedback on my `Bash scripts`!







