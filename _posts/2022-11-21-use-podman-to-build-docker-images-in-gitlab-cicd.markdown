---
layout: post
author: Cristina Negrean
title: "How I got rid of privileged mode Docker-in-Docker in GitLab CI/CD jobs with Podman"
image: /img/container_builds_with_podman.png
tags: [Container Image, Docker Image, Container Build Tools, Docker-in-Docker, Podman, GitLab CI/CD, GitLab runner, Kubernetes Executor, Kubernetes Security]
category: Cloud Native Development
---
## Background

<em>Docker-in-Docker</em> requires [privileged mode](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities) to function, which is a significant security concern.
Docker deamons have root privileges, which makes them a preferred target for attackers.
In its earliest releases, <em>Kubernetes</em> offered compatibility with one container runtime: <em>Docker</em>.

In the context of <em>GitLab CI/CD jobs</em> which build and publish Docker images to a container registry, 
docker commands in scripts (<em>build.sh</em>, <em>publish.sh</em>, <em>promote.sh</em>, <em>hotfix.sh</em>) 
referenced in your pipeline definition (<em>.gitlab-ci.yml</em> file) might seem like an obvious choice.
At least that's what I've encountered in a recent assignment, doing CI/CD with GitLab and deploying services to Kubernetes (AWS EKS).
In order to enable docker commands in your CI/CD jobs, one option at hand is using [Docker-in-Docker](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker), note the inclusion of the <em>docker:19.03.12-dind</em> service in <em>.gitlab-ci.yml</em> file:

```yaml
build:
  stage: build
  image: xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-builder:1.1.1
  services:
    - docker:19.03.12-dind 
  script:
  - deployment/build.sh
```
[Listing 1 - Snippet from <em>subscriptions/.gitlab-ci.yml</em> file: build step with `docker:dind` service]

```textmate
FROM ibm-semeru-runtimes:open-17.0.5_8-jdk-jammy

# Install docker
RUN curl -fsSL get.docker.com -o get-docker.sh && sh get-docker.sh
```
[Listing 2 - Snippet from <em>subscriptions/deployment/docker/java-builder/Dockerfile</em> used to build Docker image having tag `1.1.1` that has docker tool installed. Job script will be run in the context of this image, see also <em>build</em> container image in screenshot below] 

When you have an existing <em>Kubernetes cluster for tooling</em> and <em>need to scale your build infrastructure</em>,
the GitLab Runner will run the jobs on the tooling Kubernetes cluster. In this way, each job will have its own pod.
This Pod is made up of, at the very least, a build container, a helper container, and an additional container for 
each service defined in the <em>.gitlab-ci.yml</em> or <em>config.toml</em> files. 

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/dind-alternative-podman/runner_containers_with_dind.png" alt="k9s shell dedicated runner pod for a CI CD job"/>

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Definition from [GitLab](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#use-docker-in-docker): "Docker-in-Docker" (dind) means:
* Your registered [GitLab runner](https://docs.gitlab.com/runner/install/) uses the [Docker executor](https://docs.gitlab.com/runner/executors/docker.html) or the [Kubernetes executor](https://docs.gitlab.com/runner/executors/kubernetes.html).
* The executor uses a container image of Docker, provided by Docker, to run your CI/CD jobs.

```yaml
config.template.toml:   |
      [[runners]]
        [runners.kubernetes]
          image = "docker:git"
          privileged = true

```
[Listing 3 - Snippet from initial <em>build/subscriptions-gitlab-runner</em> Kubernetes <em>ConfigMap</em>
The <em>--privileged</em> flag gives all capabilities to the container, and it also lifts all the limitations enforced by the device cgroup controller.
In other words, the container can then do almost everything that the host, hereby Kubernetes worker node, can do. This flag exists to allow special use-cases, like running Docker-in-Docker.]

When editing above ConfigMap by setting <em>privileged = false</em>, and then restarting the <em>subscriptions-gitlab-runner</em> deployment, the pipeline jobs would start 
failing with message a connection could not be established to the Docker daemon.

The Kubernetes project also deprecated Docker as a container runtime after v1.20, and `containerd` became the default container runtime since v1.22. 
The `dockershim` component of Kubernetes, that allows to use Docker as a container runtime for Kubernetes, [was removed in release v1.24](https://kubernetes.io/docs/tasks/administer-cluster/migrating-from-dockershim/check-if-dockershim-removal-affects-you/#find-docker-dependencies).

The privileged mode required for running <em>docker:dind</em>, as well as complying with Kubernetes cluster security guidelines on keeping components up-to-date,
are pushing towards [finding alternatives for Docker commands in the context of GitLab CI/CD jobs](https://docs.gitlab.com/ee/ci/docker/using_docker_build.html#docker-alternatives).

## Kubernetes and Docker Don't Panic: Container Build Tool Options Galore

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Reading the [Kubernetes documentation](https://kubernetes.io/blog/2020/12/02/dont-panic-kubernetes-and-docker/) it became fast obvious,
I could still keep using <em>Dockerfile's</em>, the textual definition of how to build container images, I just needed to swap the build tool to one that is <em>rootless by design</em>: 
"The image that Docker produces isn’t really a Docker-specific image—it’s an OCI (Open Container Initiative) image. 
Any OCI-compliant image, <em>regardless of the tool you use to build it</em>, will look the same to Kubernetes. Both containerd and CRI-O know how to pull those images and run them." 

My process of choosing a <em>rootless by design</em> container build tool started with making an inventory of available open source options: 
* Google's [kaniko](https://github.com/GoogleContainerTools/kaniko) and [jib](https://github.com/GoogleContainerTools/jib)
* [Red Hat's](https://developers.redhat.com/blog/2019/02/21/podman-and-buildah-for-docker-users#) [podman](https://podman.io/) and [buildah](https://buildah.io/) 
* [Cloud Native Buildpacks](https://buildpacks.io/) and [Paketo](https://paketo.io/) from the [CNCF](https://www.cncf.io/) projects landscape 

I gave up on <em>kaniko</em> after I've stumbled on [this performance issue report](https://github.com/GoogleContainerTools/kaniko/issues/875),
and one colleague from Operations asked me whether I ran into [this error](https://github.com/GoogleContainerTools/kaniko/issues/2214),
which was fixed by removing the mail symbolic loop.
One of the pipelines I needed to refactor was pretty slow already, thus I was looking for an option at least equally performant or faster compared to <em>dind</em>.

Next I dived into <em>jib</em> which takes a Java developer friendly approach with no Dockerfile to maintain and provides plugins to integrate with Maven or Gradle.
I've started a proof-of-concept (POC) on the simpler pipeline I needed to refactor, a Java Spring Boot application for AWS Simple Queue Service (SQS) dead-letter-queue management, building code with Gradle tool. 
In the POC, first step was to build the container image with <em>jib</em> Gradle plugin, and then publish it to Amazon Elastic Container Registry (ECR), 
using the [easiest, most straight forward authentication method](https://github.com/GoogleContainerTools/jib/blob/master/jib-maven-plugin/README.md#using-specific-credentials):

```textmate
...
plugins {
    id 'java'
    id 'org.springframework.boot' version '2.7.5'
    ...
    id "com.google.cloud.tools.jib" version "3.3.1"
}
...

apply plugin: "com.google.cloud.tools.jib"

springBoot {
    bootJar {
        layered {
            // application follows Boot's defaults
            application {
                intoLayer("spring-boot-loader") {
                    include "org/springframework/boot/loader/**"
                }
                intoLayer("application")
            }
            // for dependencies we also follow default
            dependencies {
                intoLayer("dependencies")
            }
            layerOrder = ["dependencies", "spring-boot-loader", "application"]
        }
        buildInfo {
            properties {
                // ensure builds result in the same artifact when nothing's changed:
                time = null
            }
        }
    }
}

jib.from.image='xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-service:2.3.0'
jib.to.image='xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/subscriptions-sqs-dlq-mgmt'
jib.from.auth.username='AWS'
jib.to.auth.username='AWS'
jib.to.auth.password='***************************'
jib.from.auth.password='***************************'
jib.to.tags = ['build-jib-latest']
jib.container.ports=['8080']
jib.container.user='1000'
```
[Listing 4 - `build.gradle` snippet exemplifying jib integration with obfuscated AWS account in ECR repositoryUri, obfuscated ECR password]

```textmate
➜  subscriptions-sqs-dlq-mgmt (SUBS-3835-JIB) ✗ ./gradlew jib

> Task :jib

Containerizing application to xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/subscriptions-sqs-dlq-mgmt, xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/subscriptions-sqs-dlq-mgmt:build-jib-latest...
Base image 'xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-service:2.3.0' does not use a specific image digest - build may not be reproducible
Using credentials from to.auth for xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/subscriptions-sqs-dlq-mgmt
Executing tasks:
Executing tasks:
Executing tasks:
[==                            ] 6.3% complete
The base image requires auth. Trying again for xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-service:2.3.0...
Using credentials from from.auth for xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-service:2.3.0
Executing tasks:
Using base image with digest: sha256:943e7cfcf79149a3e549f5a36550c37d318aaf7d84bb6cfbdcaf8672a0ebee98
Executing tasks:
Executing tasks:
[==========                    ] 33.3% complete
Executing tasks:
[==========                    ] 33.3% complete
> checking base image layer sha256:223828f10a72...
Executing tasks:
[==========                    ] 33.3% complete
> checking base image layer sha256:223828f10a72...
> checking base image layer sha256:1b4b24710ad0...
Executing tasks:
[==========                    ] 33.3% complete
> checking base image layer sha256:223828f10a72...
> checking base image layer sha256:1b4b24710ad0...
Executing tasks:
[==========                    ] 33.3% complete
> checking base image layer sha256:223828f10a72...
> checking base image layer sha256:1b4b24710ad0...
> checking base image layer sha256:cf92e523b49e...
> checking base image layer sha256:0b7a45bc802a...
Executing tasks:
[==========                    ] 33.3% complete
> checking base image layer sha256:223828f10a72...
> checking base image layer sha256:1b4b24710ad0...
> checking base image layer sha256:cf92e523b49e...
> checking base image layer sha256:0b7a45bc802a...
Executing tasks:
[==========                    ] 34.7% complete
> checking base image layer sha256:223828f10a72...
> checking base image layer sha256:cf92e523b49e...
> checking base image layer sha256:0b7a45bc802a...
Executing tasks:
[===========                   ] 36.1% complete
> checking base image layer sha256:223828f10a72...
> checking base image layer sha256:0b7a45bc802a...
Executing tasks:
[===========                   ] 37.5% complete
> checking base image layer sha256:223828f10a72...
Executing tasks:
[============                  ] 38.9% complete
Executing tasks:

Container entrypoint set to [java, -cp, @/app/jib-classpath-file, nl.nederlandseloterij.subway.sqsdlqmgmt.SqsDlqMgmtApplication]
Executing tasks:
Executing tasks:
Executing tasks:

Built and pushed image as xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/subscriptions-sqs-dlq-mgmt, xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/subscriptions-sqs-dlq-mgmt:build-jib-latest
Executing tasks:
[============================  ] 91.7% complete
> launching layer pushers


BUILD SUCCESSFUL in 14s
4 actionable tasks: 3 executed, 1 up-to-date
➜  subscriptions-sqs-dlq-mgmt (SUBS-3835-JIB) ✗ 
```
[Listing 5 - `./gradlew jib` output with obfuscated AWS account in ECR repositoryUri's]

I went through documentation to get more advanced and secure ways of authenticating to Elastic Container Registry (ECR) with <em>jib</em>, and found alternatives:
* [credential helper docker-credential-ecr-login](https://github.com/GoogleContainerTools/jib/blob/master/jib-maven-plugin/README.md#using-docker-credential-helpers). This would require docker deamon, so not getting rid of <em>privileged mode</em>.
* do a <em>docker login</em> or <em>podman login</em> in a pipeline job script, to have a credential file written to [default places where Jib searches](https://github.com/GoogleContainerTools/jib/blob/master/docs/faq.md#what-should-i-do-when-the-registry-responds-with-unauthorized).

The <em>docker login</em> was a no-brainer, that was what the pipeline job script was already using, i.e.:

```textmate
aws ecr get-login-password --region eu-west-1 | docker login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions
```

or on own machine using [aws-vault](https://cristina.tech/2021/11/12/developers-lift-and-shift-to-eks):

```textmate
aws-vault exec ecr -- aws ecr  get-login-password --region eu-west-1 | docker login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions
```
[Listing 6 - <em>ECR authentication</em> in pipeline script and local machine using <em>docker login</em> with obfuscated AWS account in ECR repositoryUri]

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> <em>docker login</em> creates or updates `$HOME/.docker/config.json` file, which contains in "auths" a list of authenticated registries and the credentials store, like "osxkeychain" on macOS.

The <em>podman login</em> proved to be exactly what I was looking for. <em>Podman</em> is a open-sourced Red Hat product
designed to build, manage and run containers with a Kubernetes-like approach.
* Daemon-less - <em>Docker uses a deamon</em>, the ongoing program running in the background, to create images and run containers. <em>Podman has a daemon-less architecture</em> which means it can run containers under the user starting the container.
* Rootless - <em>Podman</em>, since it doesn't have a deamon to manage its activity, also dispenses root privileges for its containers. 
* Drop-in-replacement for docker commands: Podman provides a Docker-compatible command line front end that can simply alias the Docker cli. One catch is that for the <em>build</em> command <em>Podman behavior</em> is to call <em>buildah bud</em>. <em>Buildah</em> provides the <em>build-using-dockerfile (bud)</em> command that emulates <em>Docker's build command</em>
* both <em>Podman</em> and <em>Buildah</em> are actively maintained, after being open sourced by Red Hat, and there is a community and plenty of documentation around them. Red Hat is also using [Podman](https://www.redhat.com/en/blog/rhel-9-delivers-latest-container-technologies-development-and-production) actively in RedHat and CentOS versions 8 and higher.

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> On local machines where one has [aws-vault installed and configured](https://cristina.tech/2021/11/12/developers-lift-and-shift-to-eks), <em>podman login</em> command is: 
```textmate
aws-vault exec ecr -- aws ecr  get-login-password --region eu-west-1 | podman login --username AWS --password-stdin xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions
```
and <em>podman login</em> will create or update file `$HOME/.config/containers/auth.json`, and will then store the username and password from STDIN as a base64 encoded string in "auth" for each authenticated registry listed in "auths". See [here](https://docs.podman.io/en/latest/markdown/podman-login.1.html) <em>podman-login</em> full docs.

Thus <em>Podman</em> is a much better fit than <em>jib</em> for existing pipelines intensively using scripting with Docker commands.
I definitely recommend <em>jib</em> when starting a Java project from scratch. I've even seen <em>jib</em> successfully in action with [skaffold](https://skaffold.dev/docs/pipeline-stages/builders/jib/) which is a command line tool open sourced by Google, that facilitates continuous development for container based & Kubernetes applications.

## Unprivileged Docker/OCI Container Image Builds: Implementation using Podman

### Update build container base image

```textmate
FROM ibm-semeru-runtimes:open-17.0.5_8-jdk-jammy

RUN apt-get update && apt-get install -y podman git jq gettext unzip && rm -rf /var/lib/apt/lists/*

ENV LANG en_US.UTF-8

# Install AWS cli v2
RUN curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" \
    && unzip awscliv2.zip \
    && ./aws/install -b /bin \
    && aws --version
```
[Listing 7 - Snippet from <em>subscriptions/deployment/docker/java-builder/Dockerfile</em> used to build Docker image having tag `2.0.1` that has podman installed instead of docker]

### Refactor docker commands in job scripts to use podman

1) Authenticating to AWS ECR: simply replace `docker` with `podman`: 

```bash
echo "Login in into ECR $ECR_URL"
aws ecr get-login-password --region eu-west-1 | podman login --username AWS --password-stdin $ECR_URL
```
[Listing 8 - Snippet from <em>subscriptions/deployment/auth/auth-to-tooling-ecr.sh</em> script]

2) Build and publishing Docker container images: simply replace `docker` with `podman`:

```bash
echo $(podman --version)

EXTRA_GRADLE_OPTS="--no-daemon sonarqube -Dsonar.host.url=${SONAR_URL} -Dsonar.login=${SONAR_LOGIN_TOKEN}"
PUBLISH_FUNCTION="publish_service_ecr"

function publish_service_ecr {
  DOCKER_IMAGE_BUILD_PATH=$1;
  DOCKER_IMAGE_NAME=$2;

  echo "Building image $DOCKER_IMAGE_NAME at path $DOCKER_IMAGE_BUILD_PATH"

  BRANCH_TAG="${DOCKER_IMAGE_NAME}:${CI_COMMIT_REF_SLUG}_${CI_PIPELINE_ID}"
  LATEST_TAG="${DOCKER_IMAGE_NAME}:latest_${CI_COMMIT_REF_SLUG}"
  APP_VERSION=`cat ../VERSION`
  BUILD_TAG="${DOCKER_IMAGE_NAME}:build_${APP_VERSION}_${CI_PIPELINE_ID}"
  TAGS="-t ${BRANCH_TAG} -t ${BUILD_TAG} -t ${LATEST_TAG}"

  #copy the project root VERSION file to the docker-image for this service
  cp ../VERSION ${DOCKER_IMAGE_BUILD_PATH}/build/VERSION
  #extract the libs jars
  cd ${DOCKER_IMAGE_BUILD_PATH}/build/libs
  java -Djarmode=layertools -jar *.jar extract
  cd -
  podman build \
    --build-arg PIPELINE_ID_ARG=${CI_PIPELINE_ID} \
    ${TAGS} \
    -f docker/java-service/Dockerfile-build \
    ${DOCKER_IMAGE_BUILD_PATH}

  podman push "${BRANCH_TAG}"
  podman push "${BUILD_TAG}"
  podman push "${LATEST_TAG}"
}

#BUILD
../gradlew --build-cache --stacktrace --parallel --max-workers 4 build -p ../ $EXTRA_GRADLE_OPTS

SERVICES=$(cat services | xargs)

#PUBLISH
mkdir -p tmp

for SERVICE in $SERVICES
do
  echo "Publishing service: $SERVICE";
  #authenticate to ECR
  auth/auth-to-tooling-ecr.sh
  #call publish_service_ecr function with container image build path and image name input parameters 
  #redirect stderr and stdout to temporary log file
  $PUBLISH_FUNCTION ../subscriptions/$SERVICE/$SERVICE-app xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/$SERVICE-app 2> tmp/publish_service_$SERVICE.log > tmp/publish_service_$SERVICE.log &
done;
```
[Listing 9 - Snippet from <em>subscriptions/deployment/build.sh</em> script with obfuscated AWS account in ECR repositoryUri]

where <em>DOCKER_IMAGE_BUILD_PATH</em> is the <em>Gradle module path</em> of each application service, i.e. `./subscriptions/batch/batch-app` for `batch` (micro)service.
That is because application service Dockerfile <em>docker/java-service/Dockerfile-build</em> 
references the gradle module <em>build</em> output, with <em>SpringBoot jar layering order</em>: 
<em>dependencies</em>, <em>spring-boot-loader</em>, <em>libs-dependencies</em> and <em>application</em>.

```textmate
FROM xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-service:2.3.1

ARG PIPELINE_ID_ARG=000000
ENV PIPELINE_ID=${PIPELINE_ID_ARG}

COPY build/libs/dependencies/ ./
COPY build/libs/spring-boot-loader/ ./
COPY build/libs/libs-dependencies/ ./
COPY build/libs/application/ ./
COPY build/VERSION ./
```
[Listing 10 - <em>subscriptions/deployment/docker/java-service/Dockerfile-build</em> file with obfuscated AWS account in ECR repositoryUri. Container image `java-service` having tag `2.3.1` includes a service user and can run as non root.]

The pipeline has also <em>promote</em> and <em>hotfix</em> flows, for those the refactor was equally a drop-in replacement of 
<em>docker</em> command with <em>podman</em> command. 

The <em>deploy</em> flows are using [Helm template](https://helm.sh/docs/helm/helm_template/) 
and <em>kubectl apply</em> utilities and did not use Docker commands. 
As stated as well earlier, a Docker image isn’t really a Docker-specific image—it’s an OCI (Open Container Initiative) image.
Any OCI-compliant image, <em>regardless of the tool you use to build it</em>, will look the same to Kubernetes no matter which container runtime: containerd, CRI-O, Docker, etc. 
All container runtimes will know how to pull those (OCI) images, from AWS ECR here, and run them.

### Remove <em>docker:dind</em> sevice from <em>.gitlab-ci.yml</em>

```yaml
build:
  stage: build
  image: xxxxxxxxxxxx.dkr.ecr.eu-west-1.amazonaws.com/subscriptions/java-builder:2.0.1
  script:
  - deployment/build.sh
```
[Listing 11 - Snippet from <em>subscriptions/.gitlab-ci.yml</em> file: build step without `docker:dind` service]

As you can see in below screenshot, there is no more `svc-0` container with a `docker:dind` image, and job is run in the context of build container with new image tag `2.0.1`:

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/dind-alternative-podman/runner_containers_no_dind.png" alt="k9s shell dedicated runner pod for a CI CD job"/>

### Drop privileged mode for Kubernetes executor Gitlab runner deployment configuration

```yaml
config.template.toml:   |
      [[runners]]
        [runners.kubernetes]
          image = "ubuntu:22.04"
          privileged = false

```
[Listing 12 - Snippet from updated <em>build/subscriptions-gitlab-runner</em> Kubernetes <em>ConfigMap</em> with <em>--privileged</em> flag toggled off]

## Conclusion

As a result, getting rid of <em>dind</em>:
* facilitates upgrading Kubernetes version on tooling cluster to a version where Docker is not default runtime, by making sure no privileged <em>pods</em> execute <em>Docker commands</em>.
* improves Kubernetes cluster <em>security</em>, since dropping <em>privileged</em> mode for containers on tooling cluster worker nodes.
* facilitates Kubernetes application security, on <em>podman run</em> usage, as containers in <em>Podman</em> do not have root access by default.

## Reference Shelf

I did not reinvent the wheel, there are others which have discovered Podman as a Docker alternative before me:
* [Building Docker images on GitLab CI: Docker-in-Docker and Podman](https://pythonspeed.com/articles/gitlab-build-docker-image/)
* [Podman vs Docker: 6 Reasons why I am HAPPY I switched](https://www.smarthomebeginner.com/podman-vs-docker/)