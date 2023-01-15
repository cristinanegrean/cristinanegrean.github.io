---
layout: post
author: Cristina Negrean
title: "Elasticsearch & Kibana on Lightweight Kubernetes with Rancher Desktop"
image: /img/elasticsearch_kibana_on_k3s_with_rancher.png
tags: [ElasticSearch, Kibana, Helm, Open Source, Rancher Desktop, Kubernetes]
category: [Cloud Native Development]
---
## It’s that easy: a few clicks and you get a Lightweight Kubernetes cluster!

I needed a local sandbox for ElasticSearch and Kibana for some learnings. I could as well use a <em>docker-compose</em> file, but I thought I'd give it a try and spin it up directly in a Kubernetes sandbox. 
The only pre-requisite is having open source [Rancher Desktop](https://rancherdesktop.io/) by SUSE on your machine. It is based on [Lightweight Kubernetes (k3s)](https://k3s.io/) and after the installation you get 
a single node cluster and all the command line tools you need to deploy apps to it (kubectl/nerdctl/helm).

```textmate
➜  ~ kubectl top nodes
NAME                   CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
lima-rancher-desktop   1190m        19%    9912Mi          61%       
➜  ~ 
```

## Installing ElasticSearch

I used [Bitnami ElasticSearch](https://artifacthub.io/packages/helm/bitnami/elasticsearch) and for that it is needed that your Helm installation can search the [Bitnami](https://artifacthub.io/packages/helm/bitnami) repo:

```textmate
➜  ~ helm repo add bitnami https://charts.bitnami.com/bitnami 
"bitnami" has been added to your repositories
➜  ~ helm repo update
```
 
If you have access to multiple Kubernetes clusters, make sure to select the `rancher-desktop` context, before creating a namespace - hereby <em>elk</em> - to contain and isolate your ElasticSearch deployment,
as below:

```bash
➜  ~ kubectl config use-context rancher-desktop
Switched to context "rancher-desktop".
➜  ~ kubectl create ns elk
namespace/elk created
➜  ~  helm upgrade --install elasticsearch --namespace=elk --set master.replicaCount=3 bitnami/elasticsearch
Release "elasticsearch" has been upgraded. Happy Helming!
NAME: elasticsearch
LAST DEPLOYED: Sun Jan 15 20:36:44 2023
NAMESPACE: elk
STATUS: deployed
REVISION: 2
TEST SUITE: None
NOTES:
CHART NAME: elasticsearch
CHART VERSION: 19.5.8
APP VERSION: 8.6.0
```

Above deploys Elasticsearch with nine pods - three master replicas, 2 client/coordinating replicas to balance service requests, 2 data pods & 2 ingest replicas - using a prebuilt helm chart.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/rancher-k3s-elk/pods.png" alt="k9s ElasticSearch and Kibana pods"/>

You can then port forward in <em>k9s</em> or using `kubectl port-forward --namespace elk svc/elasticsearch 9200:9200` to test your installation:

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/rancher-k3s-elk/k9s_port_forward_elasticsearch.png" alt="k9s port forward into elasticsearch"/>

```bash
➜  ~ curl http://localhost:9200
{
  "name" : "elasticsearch-coordinating-1",
  "cluster_name" : "elastic",
  "cluster_uuid" : "IpIkCNYuReC2AEH9CqyfGA",
  "version" : {
    "number" : "8.6.0",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "f67ef2df40237445caa70e2fef79471cc608d70d",
    "build_date" : "2023-01-04T09:35:21.782467981Z",
    "build_snapshot" : false,
    "lucene_version" : "9.4.2",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
➜  ~ 
```


## Installing Kibana 

Before installing Kibana using as well a prebuilt helm chart from [Bitnami](https://artifacthub.io/packages/helm/bitnami/kibana) I've created a small manifest file named <em>kibana-values.yaml</em>.
It contains the details on how <em>Kibana</em> should reach the <em>ElasticSearch</em> installation:

```textmate
elasticsearch:
  hosts: [elasticsearch.elk.svc.cluster.local]
  port: 9200
➜  ~ 
```

```bash
➜  ~ helm upgrade --install kibana --values kibana-values.yaml --namespace elk bitnami/kibana        
Release "kibana" has been upgraded. Happy Helming!
NAME: kibana
LAST DEPLOYED: Sun Jan 15 20:47:59 2023
NAMESPACE: elk
STATUS: deployed
REVISION: 3
TEST SUITE: None
NOTES:
CHART NAME: kibana
CHART VERSION: 10.2.12
APP VERSION: 8.6.0
```

After the above installation completes you can use <em>k9s</em> or `kubectl port-forward --namespace elk svc/kibana 5601:5601` and open Kibana dashboard in your favorite browser:

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/rancher-k3s-elk/k9s_port_forward_kibana.png" alt="k9s port forward into kibana"/>

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/rancher-k3s-elk/browser.png" alt="Kibana"/>

## Conclusion

After following this tutorial, you should have Elasticsearch deployed on nine dedicated pods. The roles for the pods are defined and can be tweaked via the prebuilt Helm chart.

You should also have the open source, browser based analytics and search dashboard for Elasticsearch installed, so you can visualize data ingested in your Elasticsearch cluster.



