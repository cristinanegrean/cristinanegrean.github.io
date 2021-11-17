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
* [kubectx](https://kubernetes.io/docs/tasks/tools/)

On macOS using Homebrew package manager, dependencies installation is a one liner: 

`brew install awscli kubectl aws-vault derailed/k9s/k9s kubectx`

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
Listing 1: `cat ~/.aws/config`

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> If the development source code repo contains as well some Terraform code, you may find the AWS account numbers configured in there.

## AWS EKS Access

Now that we have set up AWS vault and the named profile, we can write a `bash script` based on `aws-vault` commands 
to setup all Elastic Kubernetes Service clusters, the operation guy or girl has set up developer access for. This is
needed first time you set up the EKS clusters, to add them to your `.kube/config` file.

```bash
#!/usr/bin/env bash

PROFILE=$1
[ -z "$PROFILE" ] && echo "You need to provide a profile name" && exit 1
REGION=${2:-eu-central-1}
aws-vault --debug exec ${PROFILE} -- aws eks update-kubeconfig --name nlo-${PROFILE}-gateway --region ${REGION}
#
```
Listing 2: `cat auth-eks-setup.sh`

Do not forget to make it executable: `chmod +x auth-eks-setup.sh`

Let's depict the Bash script and how it works:
* validates and documents its permitted usage patterns `./auth-eks-setup.sh <profile>` or `./auth-eks-setup.sh <profile> <region>`
* in case no region command line argument is given, REGION bash script variable will default to value `eu-central-1`. The default is based on the fact that all EKS clusters I have developer access to, in my current assignment, are provisioned in AWS region Europe (Frankfurt), except for the `tst` AWS account and profile, where that is `eu-west-1` aka Europe (Ireland). Hence the region is an optional command line argument, thus adapt this to your own use case.
* returns a set of temporary security credentials in a [session](https://github.com/99designs/aws-vault/blob/master/USAGE.md#managing-sessions) that you can use to access Amazon Web Services resources (Elastic Kubernetes Service is one such resource) in the `<profile>` AWS account, where profile command line argument should be one of the profiles you've defined in `~/.aws/config`. Note you most likely will not have access to any resources in the AWS account where your IAM user resides, thus the first command line argument will be: `dev`, `tst` or `acc`
* for in-depth understanding, note that `aws-vault exec` uses in the background the `AWS Security Token Service API` to generate the set of temporary credentials via the `GetSessionToken` or `AssumeRole` API calls
* the set of temporarily credentials within the `aws-vault session` is then used to authenticate, add cluster and context to your `~/.kube/config` file.  Replace hard coded strings in cluster name according to your own project naming convention.
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
```
Listing 3: `./auth-eks-setup.sh dev` with obfuscated AWS account ID of `dev` profile

After adding project related EKS clusters to your `~/.kube/config` file, use `kubectx` to assign contexts shorter names.
Example:

```bash
kubectx nlo-dev-gateway=arn:aws:eks:eu-central-1:10xxxxxxxxx3:cluster/nlo-dev-gateway
kubectx nlo-tst-gateway=arn:aws:eks:eu-west-1:15xxxxxxxxx0:cluster/nlo-tst-gateway
kubectx nlo-acc-gateway=arn:aws:eks:eu-central-1:80xxxxxxxxx5:cluster/nlo-acc-gateway
```
Listing 4: renaming cluster contexts for Dutch Lottery API Gateway assignment with obfuscated AWS account IDs

Above `kubectx` commands, will alter automatically your `~/.kube/config` file, giving an alias kind of `name` to the original
context, that is much easier and shorter.

```
contexts:
- context:
    cluster: arn:aws:eks:eu-central-1:80xxxxxxxxx5:cluster/nlo-acc-gateway
    user: arn:aws:eks:eu-central-1:80xxxxxxxxx5:cluster/nlo-acc-gateway
  name: nlo-acc-gateway
- context:
    cluster: arn:aws:eks:eu-central-1:10xxxxxxxxx3:cluster/nlo-dev-gateway
    user: arn:aws:eks:eu-central-1:10xxxxxxxxx3:cluster/nlo-dev-gateway
  name: nlo-dev-gateway
- context:
    cluster: arn:aws:eks:eu-west-1:15xxxxxxxxx0:cluster/nlo-tst-gateway
    user: arn:aws:eks:eu-west-1:15xxxxxxxxx0:cluster/nlo-tst-gateway
  name: nlo-tst-gateway
```
Listing 5: `contexts` snippet example from my `~/.kube/config` file with obfuscated AWS account IDs

The last important step is to edit your `~/.kube/config` file, wrap the `get-token` around the `aws-vault` command:

```
users:
- name: arn:aws:eks:eu-central-1:10xxxxxxxxx3:cluster/nlo-dev-gateway
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - exec
      - dev
      - --
      - aws
      - --region
      - eu-central-1
      - eks
      - get-token
      - --cluster-name
      - nlo-dev-gateway
      command: aws-vault
      env: null
- name: arn:aws:eks:eu-west-1:15xxxxxxxxx0:cluster/nlo-tst-gateway
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - exec
      - tst
      - --
      - aws
      - --region
      - eu-west-1
      - eks
      - get-token
      - --cluster-name
      - nlo-tst-gateway
      command: aws-vault
      env: null      
- name: arn:aws:eks:eu-central-1:80xxxxxxxxx5:cluster/nlo-acc-gateway
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1alpha1
      args:
      - exec
      - acc
      - --
      - aws
      - --region
      - eu-central-1
      - eks
      - get-token
      - --cluster-name
      - nlo-acc-gateway
      command: aws-vault
      env: null   
```
Listing 6: `users` snippet example from my `~/.kube/config` file with obfuscated AWS account IDs

## K9s — the powerful terminal UI for Kubernetes and Switching Contexts Easily

Now you're all setup to connect to the Kubernetes clusters, list and switch contexts with one simple command: `k9s`
In `K9S` terminal UI issue command: `ctx` to list contexts. As `K9S` is using your `~/.kube/config` file, you will
notice it will connect by default to the `current-context` listed in your `~/.kube/config` file, in the terminal UI marked
by `(*)` context name suffix.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/developers-free-lift-to-eks/k9s_ctx.png"/>

You can then use `arrow up` or `arrow down` to navigate to other context and by pressing `Enter`, you switch context.

## Other useful K9S Navigation Commands

* `no` — Nodes;
* `ns` — Namespaces;
* `deploy` — Deployments;
* `po` — Pods;
* `svc` — Services;
* `ing` — Ingresses;
* `rs` — ReplicaSets;
* `cm` — ConfigMaps;

## Alternative if you prefer going fully command line 

In case you prefer interacting with the Kubernetes resources with `kubectl` and fully from your command line,
you can use `kubectx` to easily switch contexts:

```bash
➜  ~ kubectx
minikube
nlo-acc-gateway
nlo-dev-gateway
nlo-tst-gateway
➜  ~ kubectx nlo-dev-gateway
Switched to context "nlo-dev-gateway".
```

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
echo "PROFILE=${PROFILE} ; CONFIG=${CONFIG} ; VAL=${VAL}; DEPLOYMENT=${DEPLOYMENT}; NAMESPACE=${NAMESPACE};" >&2
aws-vault --debug exec ${PROFILE} -- aws ssm put-parameter --name /config-${PROFILE}/application/${CONFIG} --type String --value "${VAL}" --overwrite >&2
aws-vault --debug exec ${PROFILE} -- aws ssm put-parameter --name /config-${PROFILE}/${DEPLOYMENT}/${CONFIG} --type String --value "${VAL}" --overwrite >&2
aws-vault --debug exec ${PROFILE} -- kubectl rollout restart deployment.apps/${DEPLOYMENT} -n ${NAMESPACE} >&2
#
```
Listing 7: `cat ssm-config-overwrite.sh`

Usage: `./ssm-config-overwrite.sh <profile> <config-name> <config-value> <k8s-deployment-name> <k8s-namespace>`

## Other useful AWS SSM Commands

* describe all parameters on `tst` environment for `game-process` application service: `aws-vault exec tst -- aws ssm describe-parameters --parameter-filters "Key=Name,Option=BeginsWith,Values=/config-tst/game-process/"`
* describe all parameters on `dev` environment that contain `cruks` in parameter name: `aws-vault exec dev -- aws ssm describe-parameters --parameter-filters "Key=Name,Option=Contains,Values=cruks"`
* get parameter by name on `acc` environment and decrypt value for `SecureString` parameter type: `aws-vault exec acc -- aws ssm get-parameter --name "/config-acc/player-process/cruks-service.keystore-password" --with-decryption`