---
layout: post
author: Cristina Negrean
title: "Helm your way to Kubernetes with Spring Boot Admin"
image: /img/spring_boot_admin.png
tags: [Helm, Kubernetes, Java, AWS EKS, Spring Boot Admin, Spring Cloud]
category: Cloud Native Development
---

In this blog post I am experimenting with [Helm, the package manager for Kubernetes](https://helm.sh/) and packaging
of [codecentric's Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) for out-of-the-box *real-time insights*
and management capabilities of a suite of microservices deployed to Amazon Web Services, using AWS's container orchestration support (EKS).

One of the easiest ways to install Java web services on Amazon Elastic Kubernetes Service (EKS) is using [Helm](https://helm.sh/)
as it offers:
* package management via a common `blue-print` called `helm chart` which is a collection of `yaml` files bundled together 
* templating of dynamic configuration, think of possible Java options (`java -X`), service name, http port, any other configuration eligible for being externalised in order to make the `helm chart` more re-usable, as well as application secrets
* easy override of dynamic configuration per environment and service. Example increase instances count and maximum Java heap size for a critical service in an environment under higher load:
```yaml
replicaCount: 3
app:
  javaOpts: '-Xmx192m -Xms128m'
```
* release management by providing a history of charts installation within the EKS cluster
```bash
➜  helm-charts (master) ✗ helmv3 history spring-boot-admin
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Mon May  9 11:05:36 2022        superseded      spring-boot-admin-0.0.2 2.6.6           Install complete
2               Thu May 12 13:47:23 2022        superseded      spring-boot-admin-0.0.3 2.6.7           Upgrade complete
3               Thu May 12 15:02:49 2022        deployed        spring-boot-admin-0.0.3 2.6.7           Upgrade complete
➜  helm-charts (master) ✗ helmv3 rollback spring-boot-admin 1
Rollback was a success! Happy Helming!
➜  helm-charts (master) ✗ helmv3 history spring-boot-admin   
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Mon May  9 11:05:36 2022        superseded      spring-boot-admin-0.0.2 2.6.6           Install complete
2               Thu May 12 13:47:23 2022        superseded      spring-boot-admin-0.0.3 2.6.7           Upgrade complete
3               Thu May 12 15:02:49 2022        superseded      spring-boot-admin-0.0.3 2.6.7           Upgrade complete
4               Thu May 12 16:05:42 2022        deployed        spring-boot-admin-0.0.2 2.6.6           Rollback to 1   
➜  helm-charts (master) ✗
```

## Creating the Spring Boot Admin Helm Chart 

One of the reasons `Helm` is so popular is the ease of finding charts in open-sourced repositories. I even 
started my experiment with [this Spring Boot Admin chart](https://github.com/evryfs/helm-charts/tree/master/charts/spring-boot-admin) but then discovered
that my installation use-case required a bit different Kubernetes objects to be created:
* `ClusterRole` instead of a `Role` 
* `ClusterRoleBinding` instead of a `RoleBinding`
* I've wanted to customize the `ServiceAccount` [annotations](https://gitlab.com/cristinanegrean/helm-charts/-/blob/main/spring-boot-admin/templates/_helpers.tpl) 
to something in the trend: `eks.amazonaws.com/role-arn: arn:aws:iam::awsAccountId:role/projectName/awsEnv/eks-spring-boot-admin`
* as well as I've changed to the `ConfigMap` and `Deployment` slightly, mostly as I was experiencing problems seeing configuration being picked up by the Admin application, using [out-of-the-box helm chart](https://github.com/evryfs/helm-charts/tree/master/charts/spring-boot-admin).

I've kept the container image to the one published into [Red Hat's Quay.io registry](https://quay.io/): `quay.io/evryfs/spring-boot-admin:2.6.7`, as I:
* liked that the [maintainer](https://github.com/evryfs/spring-boot-admin) is actively providing new versions in alignment with [codecentric's Spring Boot Admin releases](https://github.com/codecentric/spring-boot-admin/tags)
* base image is using Java 17, at the moment of writing, just like the Spring Boot applications I am scraping for insights in my assignment, as part of my development experiment
* find it packages the minimum of dependencies [spring-boot-admin-starter-server](https://mvnrepository.com/artifact/de.codecentric/spring-boot-admin-starter-server/2.6.7) 
and [spring-cloud-kubernetes-fabric8-discovery](https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-kubernetes-fabric8-discovery/2.1.2) 
* wanted to keep it simple while researching the capabilities of the Admin Console

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Additional to Google search you can use `helm repo add` to expand your list of trusted helm charts repositories and `helm search repo` to search a chart in those repositories. Note: I am running both Helm versions 2 and 3 on my machine, hence my Helm 3 installation is under executable name `helmv3`
```bash
➜  ~ helmv3 repo add bitnami https://charts.bitnami.com/bitnami
"bitnami" has been added to your repositories
➜  ~ helmv3 search repo spring
NAME                         	CHART VERSION	APP VERSION	DESCRIPTION                                       
bitnami/spring-cloud-dataflow	9.0.0        	2.9.4      	Spring Cloud Data Flow is a microservices-based...
```

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Updated Helm chart can be found [here](https://gitlab.com/cristinanegrean/helm-charts)

## Lessons Learnt and Chart Usage

### Lesson 1: Namespace Bound or Cluster-Wide?

In my assignment, I needed to scrape Spring Boot Applications being installed in two Kubernetes namespaces. 
With the initial Helm chart installation and setting configuration `spring.cloud.kubernetes.discovery.all-namespaces` to `true` (see Listing 1 below), the installation is discovering only services installed
in namespace `gateway-private`, the namespace I use by project convention for all deployments of services that are not publicly exposed via an `Ingress` or `IngressRoute`.
It was not discovering any of the Spring Boot applications deployed in `gateway-public`. 
The reason for that `Roles` in Kubernetes are scoped, either bound to a specific namespace or cluster-wide. While a namespace bound `Role` is a safer practice,
my only solution for the Admin Tool to discover all the services I was interested in, while creating a `Role` instead of a `ClusterRole` is to install the `Role` and `RoleBinding` Kubernetes objects (see Listing 3 below) in both namespaces.
In which case I also need to install the `ServiceAccount` (see Listing 4 below) in both namespaces, as `User accounts` are for humans and `Service accounts` are for processes in Kubernetes, which run in pods.
`User accounts` are intended to be global. `Service accounts` are namespaced. As the `ServiceAccount` is then used in the `Deployment`, I end up,
installing the whole Helm chart in both `gateway-private` and `gateway-public` namespaces. That looks in my terminal something in the trend:

```
helm-charts (main) ✔ helmv3 install --set namespace=gateway-private \
                     --set multi.namespaced=false \
                     spring-boot-admin-gw-private ./spring-boot-admin --debug 
```

followed by:

```
helm-charts (main) ✔ helmv3 install --set namespace=gateway-public \
                     --set multi.namespaced=false \
                     spring-boot-admin-gw-public ./spring-boot-admin --debug 
```

Now I have two Kubernetes services I need to port forward into, use separate local ports and two Spring Boot Admin UI browser tabs open to be able to see all Services insights.
That is more administration work, and my requirement was to collect insights from all Spring Boot Java applications in one overview.
I ended up using the alternative solution, which is to create a `ClusterRole` and `ClusterRoleBinding` (see Listing 2 below) and leveraging chart usage:

```
helm-charts (main) ✔ helmv3 install --set namespace=gateway-private \
                     --set multi.namespaced=true \
                     spring-boot-admin ./spring-boot-admin --debug 
```

```yaml
---
# Source: spring-boot-admin/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: RELEASE-NAME-config
  labels:
    app: RELEASE-NAME
  namespace: gateway-private
data:
  application.yml: |-
    spring:
      application:
        name: admin-server
      boot:
        admin:
          context-path: '/admin'
          ui:
            title: 'Gateway Spring Boot Admin'
            brand: 'Gateway Spring Boot Admin'
      server:
        port: 8080
      logging:
        level:
          org.springframework.cloud.kubernetes: TRACE
          de.codecentric.boot.admin.discovery.ApplicationDiscoveryListener: DEBUG
      cloud:
        kubernetes:
          discovery:
            all-namespaces: true
            service-labels:
              type: gateway-base
            catalog-services-watch:
              enabled: true
              catalogServicesWatchDelay: 10000
---
```
[Listing 1 - `ConfigMap` object definition]

```yaml
---
# Source: spring-boot-admin/templates/cluster-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: RELEASE-NAME-cluster-role
  labels:
    app: RELEASE-NAME
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - pods
      - endpoints
      - namespaces
    verbs:
      - get
      - list
      - watch
---
# Source: spring-boot-admin/templates/cluster-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: RELEASE-NAME
  labels:
    app: RELEASE-NAME
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: RELEASE-NAME-cluster-role
subjects:
  - kind: ServiceAccount
    name: RELEASE-NAME-service-account
    namespace: gateway-private
---
```
[Listing 2 - `ClusterRole` and `ClusterRoleBinding` objects definition]

```yaml
---
# Source: spring-boot-admin/templates/role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: RELEASE-NAME-role
  labels:
    app: RELEASE-NAME
  namespace: gateway-private
rules:
  - apiGroups:
      - ""
    resources:
      - services
      - pods
      - endpoints
    verbs:
      - get
      - list
      - watch
---
# Source: spring-boot-admin/templates/rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: RELEASE-NAME
  labels:
    app: RELEASE-NAME
  namespace: gateway-private
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: RELEASE-NAME-role
subjects:
  - apiGroup: ""
    kind: ServiceAccount
    name: RELEASE-NAME-service-account
    namespace: gateway-private
---
```
[Listing 3 - `Role` and `RoleBinding` objects definition]

```yaml
---
# Source: spring-boot-admin/templates/serviceaccount.yaml
kind: ServiceAccount
apiVersion: v1
metadata:
  name: RELEASE-NAME-service-account
  labels:
    app: RELEASE-NAME
  namespace: gateway-private
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::[awsAccountId]:role/gateway/dev/eks-spring-boot-admin
---
```
[Listing 4 - `ServiceAccount` object definition]

### Lesson 2: Filter Out Noise

In each of the two namespaces, there is more running than just Spring Boot applications, so I needed to define a filter for which services to scrape.
This can be done using property `spring.cloud.kubernetes.discovery.service-labels` (see Listing 1 above), a map is required here for label name and value.

Notice the last column of the listing below. You can be more creative than me in finding a better Kubernetes service label in your project 🎉

```
NAME                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE     TYPE
eventhub-system              ClusterIP   X.X.X.X          <none>        80/TCP            350d    gateway-base
game-process                 ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
game-system                  ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
inlane-process               ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
marketing-system             ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
player-process               ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
player-system                ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
responsible-gaming-system    ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
salesforce-sync-system       ClusterIP   X.X.X.X          <none>        80/TCP            360d    gateway-base
spring-boot-admin            ClusterIP   X.X.X.X          <none>        8080/TCP          7h25m   
subscription-process         ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
subscription-sales-process   ClusterIP   X.X.X.X          <none>        80/TCP            557d    gateway-base
traefik-ingress-private      ClusterIP   X.X.X.X          <none>        80/TCP,8200/TCP   567d    
```
[Listing 5 - `Service` objects within `gateway-private` namespace]

As you can see in the `Wallboard` below, there are no `spring-boot-admin` or `traefik-ingress-private` services being discovered.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spring-boot-admin/wallboard.png" alt="Wallboard"/>

### Lesson 3: Customizations

First time, I have installed the Helm chart, I was noticing an `unspecified`, see `Wallboard` image above, but also in the `Applications` overview feature.
That is when your Spring Boot Application has no `build version` in the JSON output of `/actuator/info` endpoint. There may be valid reasons for not exposing the
`build version` ;).

So what if you would like to brand your installation of Spring Boot Admin UI a bit? Well it is possible!
Just a few examples of possible overrides in [codecentric's Spring Boot Admin](https://codecentric.github.io/spring-boot-admin/current/):
* Page-Title to be shown via property `spring.boot.admin.ui.title`. Corresponding chart template value is `ui.title`.
* Brand to be shown in the navbar via property `spring.boot.admin.ui.brand`. As this one is an image asset, it makes sense to build a custom Docker image that includes the image asset.

Below screenshot exemplifies both the `Page-Title` and `Build version` customizations.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spring-boot-admin/applications.png" alt="Applications Status Overview"/>

## Useful Features of the Admin Console

A former colleague wrote a [Trifork blog post](https://blog.trifork.com/2018/12/06/managing-spring-boot-microservices-with-spring-boot-admin-on-kubernetes/) in the past on `Spring Boot Admin`.
The features he describes in his blog post, section `The top cool features we like and use most` are relevant as well in my current assignment, except the `DB migrations`.
To extend on [previous Trifork blog post](https://blog.trifork.com/2018/12/06/managing-spring-boot-microservices-with-spring-boot-admin-on-kubernetes/) there are a few other unmentioned features:
* be able to download a `Heap Dump` or `Thread Dump`
* list scheduled tasks

`Happy Helming` and feel free to drop me message if you've found another cool use-case for `Spring Boot Admin`!





