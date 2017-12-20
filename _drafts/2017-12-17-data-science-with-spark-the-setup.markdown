---
layout: post
author: Cristina Negrean
title: 'Data Science with Spark: The Setup'
image: /img/spark-data-science-setup.png
tags: [Data Science, PySpark, Jupyter Notebook, Databricks, virtualenv, Docker]
category: Apache Spark setup on macOS
---

I have recently completed a data science with Apache Spark training and in order to maximise it, I've decided to write a few blog posts. In this article I will show how to setup a macOS machine for test driving Apache Spark with Python, PySpark and Jupiter Notebook.

## Apache Spark and PySpark

Apache Spark is a must for Big data’s hackers. In a few words, Spark is a fast and powerful framework that provides an API to perform massive distributed processing over resilient sets of data. While using Spark, most data engineers recommends to develop either in Scala (which is the “native” Spark language) or in Python through complete PySpark API.

I've learnt that using Python against Apache Spark comes with a performance overhead over Scala but the significance depends on what you are doing. Nonetheless, Python is the perfect language for prototyping data science/machine learning fields.

## Jupyter Notebook

Jupyter Notebook is a popular application that enables you to edit, run and share Python code into a web view.

## Install Spark

To install Spark, make sure you have Java 8 or higher installed on your computer. Then, visit the [Spark downloads page](http://spark.apache.org/downloads.html). Select the latest Spark release, a prebuilt package for Hadoop, and download it directly.

cp /Users/cristina/Downloads/spark-2.2.1-bin-hadoop2.7.tgz /opt/
cd /opt/
tar -xzf spark-2.2.1-bin-hadoop2.7.tgz

Create a symbolic link:
$ ln -s spark-2.2.1-bin-hadoop2.7 /opt/spark
This way, you will be able to download and use multiple Spark versions.

Finally, tell your bash where to find Spark. To do so, configure your $PATH variables by adding the following lines in your .bash_profile file:

export SPARK_HOME=/opt/spark
export PATH=$SPARK_HOME/bin:$PATH

## Virtual Python Environment builder for PySpark data science project with virtualenv

[virtualenv](https://pypi.python.org/pypi/virtualenv) is a tool to create isolated Python environments.

First setup a project path:

```
$ export PY_WORKON_HOME=~/pyEnvs
$ mkdir -p $PY_WORKON_HOME
$ cd $PY_WORKON_HOME
```

Then if you've decided to run your PySpark project with Python2, you can install [virtualenv](https://pypi.python.org/pypi/virtualenv) with pip:

```
$ pip install --upgrade virtualenv
$ virtualenv pySpark-env
```

Python3 has a built-in support for virtual environments - venv: ```python3 -m venv $PY_WORKON_HOME/pySpark-env```

In order to active virtual environment, type: ```source pySpark-env/bin/activate```

You should be now all setup with a clean, isolated Python environment. To complete the setup, install packages and start Jupyter Notebook on the port of your choice (hereby 8988)

```
$ pip install pyspark
$ pip install findspark
$ pip install jupyter
$ pip install sklearn
$ pip install numpy
$ pip install pandas
$ pip install matplotlib
$ jupyter notebook --port 8988
```
In order to tear down the virtual environment, just type: ```deactivate```

## Virtual Python Environment builder for PySpark data science project with Docker

[To check out how this works compared to virtualenv](https://medium.com/@suci/running-pyspark-on-jupyter-notebook-with-docker-602b18ac4494)

## Code sample

## Scala Notebook for Apache Spark with Databricks
