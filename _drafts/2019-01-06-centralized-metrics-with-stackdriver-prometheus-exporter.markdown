---
layout: post
author: Cristina Negrean
title: 'Centralized metrics with Stackdriver Prometheus Exporter'
image: /img/metrics.png
tags: [Stackdriver Prometheus Exporter, Open Source, Monitoring and Alerting, Cloud Native]
category: Cloud Native Development
---
## Background

Do you package your web applications in Docker container images and run those in Google Kubernetes Engine cluster? Then most likely your application depends on at least a few GCP services. As you might then already know, Google does not currently charge for [monitoring data](https://cloud.google.com/stackdriver/pricing) when it comes to [GCP metrics](https://cloud.google.com/monitoring/api/metrics_gcp). Therefore, if you want to monitor operations of Google Cloud Platform (GCP) services, such as: Compute Engine, Cloud SQL, Cloud Dataflow and Cloud Pub/Sub, using Google's Cloud Monitoring service - [Stackdriver](https://cloud.google.com/stackdriver/) - is a perfect choice.

Above mentioned free allotment does not apply to your application custom metrics. Besides if you write applications in Java, on top of [Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html), you might as well have heard of the vendor-neutral application metrics facade [Micrometer](http://micrometer.io/). Micrometer provides built-in support for the top listed [full-metrics solution for Kubernetes](https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring/#full-metrics-pipelines): [Prometheus](https://prometheus.io/). [Prometheus](https://prometheus.io/) happens to be as well open-source, community driven, and while Google claims that Stackdriver won’t lock developers into using a particular cloud provider, a read through their [Stackdriver Kubernetes Monitoring](https://cloud.google.com/monitoring/kubernetes-engine/) documentation page currently states `Beta release` and limited to `GKE only` and something in the trend of add-on and opt-in.

An other interesting fact about Google's approach to monitoring, is that they just recently open-sourced [OpenCensus](https://opencensus.io/), a new set of libraries for collecting metrics and distributed traces, which provides a way to expose Java application services metrics in [Prometheus data format](https://opencensus.io/exporters/supported-exporters/java/prometheus/#creating-the-exporter). So one could wire [OpenCensus](https://opencensus.io/) into their Java application to expose an HTTP endpoint `metrics` for [Prometheus](https://prometheus.io/) to scrape, similarly to [Spring Boot's Actuator Prometheus endpoint](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-metrics.html#production-ready-metrics-export-prometheus) use-case. That being said, [Stackdriver](https://cloud.google.com/stackdriver/) is still unable to handle metrics in Prometheus data-format and you would need to use [Stackdriver's Prometheus integration](https://cloud.google.com/monitoring/kubernetes-engine/prometheus) - currently limited to 1,000 Prometheus-exported metric descriptions per GCP project - in order to centralize all metrics from your GCP services and Kubernetes resources, including your containerized Java applications, residing inside the Kubernetes pods.

## Installing Stackdriver Prometheus Exporter in your Kubernetes Cluster

At [Trifork](https://trifork.com/), as part of the [weareblox project](http://preview.weareblox.com/markets), I've faced the requirement of making metrics for various GCP services - Cloud SQL, Google's Dataflow and Pub/Sub - available to [Prometheus](https://prometheus.io/) and I thought it might be interesting to share the steps I've taken.

>  `Prerequisites`: This blog post assumes that you already have access to the Google Cloud Platform.
If you don't, there is a [GCP Free Tier](https://cloud.google.com/free/docs/gcp-free-tier) which gives you free resources to learn about Google Cloud Platform (GCP) services. It's possible to create a sandbox GCP project and enable Google Cloud Monitoring, by logging in to [Stackdriver](https://app.google.stackdriver.com/) and adding a workplace for your GCP project.
When you create a GCP project from scratch, you need:
* a Kubernetes cluster, that can be created from the [Google Console](https://console.cloud.google.com/kubernetes)
* a data processing pipeline, checkout [Dataflow Word Count Tutorial](https://console.cloud.google.com/dataflow?walkthrough_tutorial_id=dataflow_index) as an example
Other tooling that needs to be installed:
* [Google Cloud SDK](https://cloud.google.com/sdk/). On macOS, you can verify your installation and setup with ```gcloud config configurations list```
* [Helm](https://helm.sh/) installed. Installation instructions can be found [here](https://docs.helm.sh/using_helm/#installing-helm). <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png"/>Before installing the server portion of Helm, `tiller`, make sure you're authenticated and connected to the right Kubernetes cluster: the one belonging to the GCP project for which you want the Stackdriver metrics to be made available in Prometheus. To authenticate to a Kubernetes cluster, you can use command: ```gcloud container clusters get-credentials <your-cluster-name>```.
* [Prometheus]()

### 1. Decide what GCP metric types you want to export

### 2. Add new Service Monitor to Prometheus

### 3. Create insights and alerts in Prometheus
