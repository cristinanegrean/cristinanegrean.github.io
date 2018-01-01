---
layout: post
author: Cristina Negrean
title: 'Data Science with Spark Test Drive'
image: /img/spark-data-science-setup.png
tags: [Data Science, XebiaLabs, PySpark, Jupyter Notebook, Databricks, virtualenv]
category: Data science for software engineers
---

One of the perks of working for [XebiaLabs](https://xebialabs.com/products/xl-deploy) is our training allowance: as a software engineer you get a generous budget and 5 training days for your personal development. You're free to spend it on whatever will benefit you: internal or external trainings, conferences, as well as massive open online courses.

I've chosen to use my 2017 bound budget on an [Apache Spark Data Science training](https://training.xebia.com/data-science/data-science-with-spark/) hosted by [GoDataDriven](godatadriven.com). In retrospective, aside from the fact that I've found the trainers knowledgeable and the material well-rounded, `hereby the top reasons why I think software engineers can benefit too from immersing themselves into data science and Spark™ technology`.

### 1. Artificial intelligence is everywhere, get to know the open source code behind it.

["AI has always been one of the most exciting applications of big data and Apache Spark."](https://databricks.com/blog/2017/12/06/spark-summit-is-becoming-the-spark-ai-summit.html) -  Matei Zaharia, the creator of [Apache Spark™](https://spark.apache.org/)

Whether I am thinking of smart planning algorithms to brake world records of the famous travelling salesman problem (TSP), predictive distribution models, better diagnostic algorithms in healthcare, it is difficult to escape the reality of how much data science is picking up ground across all industries.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/big_data_landscape_2017_open_source.png"/>

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/sparkml-vs-mllib-vs-sklearn.png"/>

[Spark's Machine Learning Library](https://spark.apache.org/mllib/) was covered on the `second day` of training. For me this was also the day with the steepest learning curve and loads of machine learning concepts: `(stochastic) gradient descent`, `random decision forests in classification`, `Gaussian mixtures (GMMs)`, `n-Gram-based classification`, `statistical fit, overfitting and underfitting data in machine learning algorithms`, `linear regression`, `alternating least squares method for collaborative filtering, matrix factorisation in recommender systems`.

After this training, I still consider myself very far away from being a proficient data scientist. Nonetheless, I am informed on what's out there to be leveraged as open source tools and I've heard of the concepts and algorithms to be further mastered when implementing particular AI use-cases.

### 2. So what? Data scientist and software engineer are two different disciplines

I agree that specialising in a particular discipline and set of technologies (X-functional) is better than being a `jack of all trades`, thus above statement is justified. However reality is never that `black or white` in a such dynamic field as `technology`.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/jack_of_all_trades.png"/>
The only constant is change, continuous improvement. And improvement may require also a certain degree of experimentation by seizing opportunities to learn new skills.

### 3. Apache Spark is a must for `big data` hackers.

Like so many controversial topics, the difference between “Big” data and lower-case data might be best explained with a joke:

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> "If it fits in RAM, it’s not Big Data." - Unknown (If you know, pass it along!)

Working at the [XL Deploy](https://xebialabs.com/products/xl-deploy) product on automating and standardising complex application deployments for enterprise scale clients, means often dealing with new `infrastructure`, depending on each client's database engine and application server requirements engineering.

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/cncf_landscape_tag.png"/>
[The `App Definition & Development` Overview at Cloud Native Landscape Project](https://raw.githubusercontent.com/cncf/landscape/master/landscape/CloudNativeLandscape_latest.png)

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Not long ago, I've been spinning up Docker containers for various SQL database engines, in order to test proper handling of native SQL syntax for the brand new HTML5 XL Deploy Analytics dashboard, across various SQL dialects. By looking at the `Cloud Native Landscape Project` extract above, Spark and Hadoop <img class="thumb" src="{{site.baseurl }}/img/buildkite-64/spark.png"/> <img class="thumb" src="{{site.baseurl }}/img/buildkite-64/hadoop.png"/> appear alongside SQL and NoSQL databases & data analytics tools. They are just for that day when your backlog task is to create SQL queries for a reporting dashboard mining data of the magnitude of `petabytes`

<img class="img-responsive" src="{{ site.baseurl }}/img/site/xl-deploy-jenkins-jcr-to-sql-migration.png" alt="XL Deploy JCR to SQL migration"/>
<img class="img-responsive" src="{{ site.baseurl }}/img/author/xldeploy_dashboard_report.png"/>

### 4. Apache Spark™ is an Engine for Large-Scale Data Processing

No doubt that Python is the perfect language for prototyping data science/machine learning fields.
However there is more to [Apache Spark™](https://spark.apache.org/) than only its [Machine Learning Library](https://spark.apache.org/mllib/).

<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/spark_training.png"/>

What I liked about days 1&3 of the training is that both trainers have drawn clear overviews on the motivation to leverage [Apache Spark™](https://spark.apache.org/) on the `engineering` side of things:

* Lightning speed of computation because data are loaded in distributed memory (RAM) over a cluster of machines, making Spark suitable for faster data-sharing needs across parallel jobs.
* Comparison to `Hadoop Map-Reduce jobs` having intermediate computation results I/O bound, thus making Hadoop <img class="thumb" src="{{site.baseurl }}/img/buildkite-64/hadoop.png"/> mostly suitable for use cases where processing times are not much of a concern: such as ETLs (Extract/Transform/Load), data consolidation, and cleansing.
* Unified approach to solve `Batch`, `Streaming`, and `Interactive` use cases  
* Highly accessible through standard APIs built in `Java`, `Scala`, `Python`, `SQL` (for interactive queries)

### 5. If you take notes, you might get hooked on notebooks

As a software engineer, I strive to use the right tool for the job. In my current project there is quite a polyglot landscape of general purpose programming languages: Java, Scala, Python, some Groovy DSL. Whenever I needed to interact with the [Python SDK for XL-Deploy](https://pypi.python.org/pypi/xldeploy-py/0.0.8) code, I've been using [IntelliJ IDEA](https://www.jetbrains.com/idea/) and its [Python Plugin](https://www.jetbrains.com/help/idea/python-plugin.html). During the training, I got introduced to working with [Jupyther Notebook](http://jupyter.org/), an open source project for web-based notebooks that enables data-driven, interactive data analytics and collaborative documents with cells of code, markdown, images etc.

The installation and usage came in two flavors:
* [Spark standalone](https://spark.apache.org/docs/latest/spark-standalone.html) with PySpark and Jupyter notebook local installation which I describe in below section `Setup your own Apache Spark, PySpark and Jupyter Notebooks Playground`
* ssh tunnel with port forwarding access to a [Google Cloud DataProc managed Spark service](https://cloud.google.com/dataproc/) which challenged the trainer to monitor and resize the cluster when everyone started to work on the [Spark Streaming API](https://spark.apache.org/streaming/) exercises on the cloud managed cluster

><img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Want interactive notebooks in Python, Scala, R and SQL and a  Mini 6GB cluster for learning Apache Spark APIs with zero installation fuss? I recommend [Apache Spark on Databricks Community edition](https://community.cloud.databricks.com)
<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/spark_on_databricks.png"/>
 >> Are you a Docker fan? You might like [Jupyter Notebook Python, Scala, R, Spark, Mesos Stack](https://github.com/jupyter/docker-stacks/tree/master/all-spark-notebook)
Wait! There are [Apache Zeppelin notebooks too](https://zeppelin.apache.org/), now I need a recommender system on notebooks 😃😄😆

## Setup your own Apache Spark, PySpark and Jupyter Notebooks Playground

Has your interest been sparked? Are you curious to test drive Apache Spark™ APIs? Did you know that Spark has a convenient shell (REPL: Read-Eval-Print-Loop) and examples to interactively learn the APIs?

There are a multitude of online resources to get your machine equipped for a test drive. Bellow I describe my own installation on macOS.

Before installing Spark, make sure you have [JDK 8](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)+, Scala 2.11+, Python 2.7.9+/3.4+ (recommended as `pip` is included). Then, visit the [Spark downloads page](http://spark.apache.org/downloads.html). Select the latest Spark release, a prebuilt package for Hadoop, and download it directly.

```bash
$ sudo cp ~/Downloads/spark-2.2.1-bin-hadoop2.7.tgz /opt/
$ cd /opt/
$ sudo tar -xzf spark-2.2.1-bin-hadoop2.7.tgz
$ ln -s spark-2.2.1-bin-hadoop2.7 /opt/spark`
```
Having the symbolic link will make the path reference shorter and you will be able to download and use multiple Spark versions. Finally, tell your bash where to find Spark. To do so, configure your $PATH variable by adding the following lines in your `.bash_profile` file:

```
export SPARK_HOME=/opt/spark
export PATH=$PATH:$SPARK_HOME/bin
```

Check that you have Java, Scala, Python added to your $PATH too:

```
export JAVA_HOME=$(/usr/libexec/java_home -v 1.8)
export JRE_HOME=$JAVA_HOME/jre
export SCALA_HOME=/opt/scala-2.12.3
export PYTHON_HOME=/usr/local/bin/python3
export PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin:$SCALA_HOME/bin:$PYTHON_HOME
```

While using Spark, most data engineers recommends to develop either in Scala (which is the 'native' Spark language) or in Python through complete PySpark API. At this point you can
check your standalone installation via Spark’s shell:

| Scala       | Python |
| ------------- |:-------------:|
| *$ spark-shell* | *$ pyspark* |

Standalone Spark installation comes also with quite some samples. For the Scala sample that computes an approximation to pi, you can just open a terminal and type: `run-example SparkPi`. You will also be able to access the Spark Web UI at: `http://localhost:4040`.

### Virtual Python Environment builder for PySpark data science project with virtualenv

[virtualenv](https://pypi.python.org/pypi/virtualenv) is a tool to create isolated Python environments.

First setup a project path:
```bash
$ export PY_WORKON_HOME=~/pyEnvs
$ mkdir -p $PY_WORKON_HOME
$ cd $PY_WORKON_HOME
```

If you're running Python 2.7.9+ (not recommended), `pip` the Python package manager is already installed, but you'll have to install [virtualenv](https://pypi.python.org/pypi/virtualenv) and then create an isolated data science Python environment, named `pySpark-env` as below:

```bash
$ python pip install --upgrade virtualenv
$ virtualenv pySpark-env
```

When running Python 3.4+ (recommended), you already have built-in support for virtual environments with - venv:

```bash
$ python3 -m venv $PY_WORKON_HOME/pySpark-env
$ source pySpark-env/bin/activate
```

The source command actives the virtual environment, thus you should be now all setup with a clean, isolated Python environment. Typing ```pip freeze``` shows at this moment no Python modules installed in the `pySpark-env`. To complete the full environment setup, install general purpose modules needed for data science projects with Python and Spark, and start Jupyter Notebook on the port of your choice (hereby 8988):

```bash
$ pip install pyspark findspark jupyter sklearn scipy numpy pandas matplotlib
$ jupyter notebook --port 8988
```

Ta da! 🎉 You're all setup to create new or import existing Python3 notebooks:
<img class="img-responsive" src="{{ site.baseurl }}/img/posts/spark/jupyter_python3_notebook.png"/>

In order to tear down the virtual environment `pySpark-env`, just type: ```deactivate``` in your terminal, after stoping the Jupyter NotebookApp web server.

## The Reference Shelf

I am not disclosing the training material, however hereby some resources that I've found helpful along the way:
* [Apache Spark Machine Learning Tutorial](https://mapr.com/blog/apache-spark-machine-learning-tutorial/)
* [Recommender  Systems](https://bugra.github.io/work/notes/2014-04-19/alternating-least-squares-method-for-collaborative-filtering/)
* [Introduction to Apache Spark on Databricks](https://community.cloud.databricks.com)
* [DZone Apache Spark Refcard](https://dzone.com/refcardz/apache-spark)
