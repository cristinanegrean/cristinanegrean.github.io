---
layout: post
author: Cristina Negrean
title: "How to mitigate Kubernetes Insecure Workload Configuration"
image: /img/runAsNonRoot.png
tags: [Container Image, Container Escape, Kubernetes Security]
category: Cloud Native Development
---
## Problem

<i>Dockerfile</i>, the magic piece of text that defines how to build a container image, is nowadays more often written by developers. 
When the developer writing it is less <i>security-minded</i>, or trained on topics as: <i>Linux Kernel and containers isolation in Kubernetes</i> and <i>how to reduce Kubernetes attack surface</i>, 
chances are pretty big your software will make it to production with a security vulnerability called in Kubernetes security literature: <b>container escape</b> or <b>container breakout</b> or <b>insecure workload configuration</b>.
A malicious user could then exploit this vulnerability in order to gain access to the <i>host Kernel</i>, in Kubernetes also called <i>worker node</i>. From there the <i>attack surface</i> is pretty large.

Even if you're using [Java Image Builder (Jib)](https://github.com/GoogleContainerTools/jib), a developer friendly container tool from Google, to containerize your Java application without a <i>Dockerfile</i>, as I've seen in a project I've consulted over the summer,
the person who introduced <i>JIB</i> in the codebase of the project, forgot to use more advance configurations like the [container user](https://github.com/GoogleContainerTools/jib/issues/1029).

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Containers under the hood sketches<img class="img-responsive" src="{{ site.baseurl }}/img/posts/kubernetes-security/containerIsolation.png" alt="Container Isolation"/>

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Container image using the root user to run PID 1 - if that process is compromised, the attacker has root in the container, and any mis-configurations become much easier to exploit.<img class="img-responsive" src="{{ site.baseurl }}/img/posts/kubernetes-security/runAsRoot.png" alt="Container Running as root"/>

## Solution

#### To mitigate container escapes, build containers running as non-root, meaning there will be an extra step to get root access from user access, to be able to break out of the container.

Below <i>Dockerfile</i> defines how to invoke <i>useradd with the -u (--uid) option</i> Linux command to create a user with name <b>subway</b> and UID <b>1000</b>.
Base image in <i>Dockerfile</i> is [IBM released](https://www.infoq.com/news/2021/10/ibm-introduces-semeru-openj9/) Semeru Runtimes based on [OpenJ9](https://www.eclipse.org/openj9/) Java Virtual Machine (JVM). 

Example is for Java 17 on Linux (Ubuntu Jammy Jellyfish)

```text
FROM ibm-semeru-runtimes:open-17.0.4.1_1-jre-jammy

RUN useradd -u 1000 subway

USER 1000
...
```
[Listing 1 - `Dockerfile` definition snippet for a base Java service image]

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Kubelet in k8s version 1.21 requires `USER uid` instead of `USER username` to avoid Error: 
<i>container has runAsNonRoot and image has non-numeric user (subway), cannot verify user is non-root</i>

#### Configure a security context for pod or container in your Kubernetes workload definition, and prevent privilege escalation.

```yaml
  template:
    spec:
      containers:
        name: payments
        securityContext:
          allowPrivilegeEscalation: false
          runAsNonRoot: true
```
[Listing 2 - Snippet from Kubernetes `Deployment` workload resource definition of Java `payments` container]

#### Verify that container is using the defined non-root user to run PID 1, by acquiring a shell to the running container.

In example at hand, pod has only one container `payments`

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/kubernetes-security/runAsNonRoot.png" alt="k9s shell"/>

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> When creating a new user, the default behavior of the useradd command is to create a group with the same name as the username, and same GID as UID.

## Reference Shelf

Above was just the tip of the iceberg. If you want to learn more on Kubernetes application security best practices and exploits, you might find useful:
* [11 Ways (Not) to Get Hacked](https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/#8-run-containers-as-a-non-root-user)
* [How to stop running containers as root](https://elastisys.com/howto-stop-running-containers-as-root-in-kubernetes/)
* [How to Stop Container Escape and Prevent Privilege Escalation](https://goteleport.com/blog/stop-container-escape-privilege-escalation/)
* [Open Web Application Security Project: 2022 Top 10 Kubernetes Security Vulnerabilities](https://github.com/OWASP/www-project-kubernetes-top-ten/blob/main/2022/en/src/K01-insecure-workload-configurations.md)
* [Snyk 10 Kubernetes Security Context settings you should understand](https://snyk.io/wp-content/uploads/10-Kubernetes-Security-Context-settings-you-should-understand.pdf)
* [Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
