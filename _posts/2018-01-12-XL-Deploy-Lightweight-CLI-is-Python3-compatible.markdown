---
layout: post
author: Cristina Negrean
title: 'XL Deploy Lightweight CLI is Python 3.x interpreter compatible now'
image: /img/xl-deploy-python-light-cli.png
tags: [XebiaLabs, XL Deploy Lightweight CLI, virtualenv]
category: XL Deploy Lightweight CLI
---

### XL Deploy Lightweight Command Line Interface

The Lightweight CLI is a Python-based command-line interface that allows you to connect to and interact with a [XL Deploy server](https://xebialabs.com/products/xl-deploy/). It provides interactive user-friendly validation and error messaging. Most of all it brings useful [XL Deploy](https://xebialabs.com/products/xl-deploy/) features from the web-based graphical user interface (GUI) to the command-line. A short introduction can be found [here](https://docs.xebialabs.com/xl-deploy/concept/xl-deploy-lightweight-cli.html).

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Lightweight XLD CLI currently provides a predefined set of commands. [XebiaLabs](https://xebialabs.com/) provides as well a [CLI](https://docs.xebialabs.com/xl-deploy/concept/getting-started-with-the-xl-deploy-cli.html) for advanced users of XL Deploy, which allows to roll your own via Python scripting.

### virtualenv

[virtualenv](https://pypi.python.org/pypi/virtualenv) is a tool to create isolated Python environments. Do you need to test the XLD lightweight CLI against different versions of the Python interpreter using one machine? Then this is the tool for you! It is included with the Python 3 installation and the usage is to provide `-m venv` module option to the Python 3.x interpreter. Are you still using Python 2.x executable? Then you'll need to install `virtualenv` via the Python package manager: `python pip install --upgrade virtualenv`

> <img class="img-responsive" src="{{ site.baseurl }}/img/site/blockquote-green-red.png" alt="Note"/> Are you still pending on installing Python 3.x and are wondering which version to pick? Did you know Python 3.4+ comes with the Python package manager `pip3` included? Hereby how to install on macOS:
```
$ brew install python3
$ brew link python3
$ python3 -m ensurepip
```
On Unix, the Python 3.x interpreter is by default not installed with the executable named python, so that it does not conflict with a simultaneously installed Python 2.x executable:
```
$ python3
Python 3.6.3 (default, Oct 4 2017, 06:09:15)
$ python
Python 2.7.10 (default, Jul 15 2017, 17:16:57)
```

### Current situation [`xldeploy-py` 0.0.8](https://pypi.python.org/pypi/xldeploy-py/0.0.8) and [`xld-py-cli` 7.5.0](https://pypi.python.org/pypi/xld-py-cli/7.5.0)

The problems with the current documentation on the [XebiaLabs site](https://docs.xebialabs.com/xl-deploy/concept/xl-deploy-lightweight-cli.html) are:
* there is no mentioning of which Python interpreter versions are supported
* "This example shows how to generate a Deployfile from a directory in the XL Deploy repository" caused questions to development whether you need to install the lightweight XLD Python CLI on same machine as the XL Deploy server. Which is definitely not a requirement. The XL Deploy server can reside on a different machine than the `xld-py-cli` client, as shown below.

```bash
CristinasMBP:dev cristina$ python3 -m venv pyLightCliVer7.5.0
CristinasMBP:dev cristina$ source pyLightCliVer7.5.0/bin/activate
(pyLightCliVer7.5.0) CristinasMBP:dev cristina$ pip3 install xld-py-cli
Collecting xld-py-cli
  Using cached xld-py-cli-7.5.0.tar.gz
Collecting xldeploy-py (from xld-py-cli)
  Using cached xldeploy-py-0.0.8.tar.gz
Collecting fire (from xld-py-cli)
Collecting requests==2.11.1 (from xldeploy-py->xld-py-cli)
  Using cached requests-2.11.1-py2.py3-none-any.whl
Collecting six (from fire->xld-py-cli)
  Using cached six-1.11.0-py2.py3-none-any.whl
Installing collected packages: requests, xldeploy-py, six, fire, xld-py-cli
  Running setup.py install for xldeploy-py ... done
  Running setup.py install for xld-py-cli ... done
Successfully installed fire-0.1.2 requests-2.11.1 six-1.11.0 xld-py-cli-7.5.0 xldeploy-py-0.0.8
(pyLightCliVer7.5.0) CristinasMBP:dev cristina$ pip3 freeze
fire==0.1.2
requests==2.11.1
six==1.11.0
xld-py-cli==7.5.0
xldeploy-py==0.0.8
(pyLightCliVer7.5.0) CristinasMBP:dev cristina$ xld version
Traceback (most recent call last):
  File "/Users/cristina/dev/pyLightCliVer7.5.0/bin/xld", line 11, in <module>
    load_entry_point('xld-py-cli==7.5.0', 'console_scripts', 'xld')()
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/pkg_resources/__init__.py", line 565, in load_entry_point
    return get_distribution(dist).load_entry_point(group, name)
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/pkg_resources/__init__.py", line 2631, in load_entry_point
    return ep.load()
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/pkg_resources/__init__.py", line 2291, in load
    return self.resolve()
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/pkg_resources/__init__.py", line 2297, in resolve
    module = __import__(self.module_name, fromlist=['__name__'], level=0)
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/xld/__init__.py", line 2, in <module>
    from xld.xld_commands import XldCommands
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/xld/xld_commands.py", line 1, in <module>
    import xldeploy
  File "/Users/cristina/dev/pyLightCliVer7.5.0/lib/python3.6/site-packages/xldeploy/__init__.py", line 1, in <module>
    from client import Client
ModuleNotFoundError: No module named 'client'
```

Error above is due to the fact that Python 3.x convention is to use [absolute imports](https://wiki.python.org/moin/PortingPythonToPy3k#relative-imports) and `xldeploy-py==0.0.8` is using relative imports.

Example:

```python
from client import Client
from config import Config
```
becomes in Python 2-3 compatible as below:

```python
from xldeploy.client import Client
from xldeploy.config import Config
```

There are more examples of Python 3 incompatible syntax with regards to `urllib2`, `urlparse`, `unitest`, `print` is now a function, `% formatting` needs to be done inside the parenthesis (it is not officially deprecated yet, however it is discouraged in favour of `str.format`) and some other.

### Python 3.x support sneak preview in `xldeploy-py` 0.0.9 and `xld-py-cli` 7.6.0

The new versions are not yet released to the public [PyPi](https://pypi.python.org/pypi) Python package index repository, so below an intro on how to install using `virtualenv`, `pip3` and locally built distributions of the Python modules. Once [`xld-py-cli` 7.6.0](https://pypi.python.org/pypi/xld-py-cli/7.6.0) is available in `PyPi`, the right version of the required Python SDK for XL-Deploy will be installed out-of-the box, so you can skip the `pip3 install xldeploy-py` step.

```bash
CristinasMBP:dev cristina$ python3 -m venv pyLightCliPy3
CristinasMBP:dev cristina$ source pyLightCliPy3/bin/activate
(pyLightCliPy3) CristinasMBP:dev cristina$ pip3 freeze
(pyLightCliPy3) CristinasMBP:dev cristina$ pip3 install /Users/cristina/dev/xldeploy-py/dist/xldeploy-py-0.0.9.tar.gz
Processing ./xldeploy-py/dist/xldeploy-py-0.0.9.tar.gz
Collecting requests==2.11.1 (from xldeploy-py==0.0.9)
  Using cached requests-2.11.1-py2.py3-none-any.whl
Installing collected packages: requests, xldeploy-py
  Running setup.py install for xldeploy-py ... done
Successfully installed requests-2.11.1 xldeploy-py-0.0.9
(pyLightCliPy3) CristinasMBP:dev cristina$ pip3 freeze
requests==2.11.1
xldeploy-py==0.0.9
(pyLightCliPy3) CristinasMBP:dev cristina$ pip3 install /Users/cristina/dev/xl-deploy/xld-py/xld-py-cli/build/distributions/xld-py-cli-7.6.0-SNAPSHOT.tar.gz
Processing ./xl-deploy/xld-py/xld-py-cli/build/distributions/xld-py-cli-7.6.0-SNAPSHOT.tar.gz
Requirement already satisfied: xldeploy-py in ./pyLightCliPy3/lib/python3.6/site-packages (from xld-py-cli===7.6.0-SNAPSHOT)
Collecting fire (from xld-py-cli===7.6.0-SNAPSHOT)
Requirement already satisfied: requests==2.11.1 in ./pyLightCliPy3/lib/python3.6/site-packages (from xldeploy-py->xld-py-cli===7.6.0-SNAPSHOT)
Collecting six (from fire->xld-py-cli===7.6.0-SNAPSHOT)
  Using cached six-1.11.0-py2.py3-none-any.whl
Installing collected packages: six, fire, xld-py-cli
  Running setup.py install for xld-py-cli ... done
Successfully installed fire-0.1.2 six-1.11.0 xld-py-cli-7.6.0-SNAPSHOT
(pyLightCliPy3) CristinasMBP:dev cristina$ pip3 freeze
fire==0.1.2
requests==2.11.1
six==1.11.0
xld-py-cli===7.6.0-SNAPSHOT
xldeploy-py==0.0.9
(pyLightCliPy3) CristinasMBP:dev cristina$ xld version
xld-py-cli 7.6.0-SNAPSHOT from /Users/cristina/dev/pyLightCliPy3/lib/python3.6/site-packages/xld (python 3.6.3)
(pyLightCliPy3) CristinasMBP:dev cristina$ xld --url http://sprintly.xebialabs.com:4516 --username yourXLDUsername --password yourXLDPass generate /Infrastructure/CICD_Pipeline
xld {
  scope(
    forInfrastructure: 'Infrastructure/CICD_Pipeline'
  ) {
    infrastructure('test_server_01', 'overthere.LocalHost') {
      os = com.xebialabs.overthere.OperatingSystemFamily.UNIX
      // no Deployfile renderer found for 'Infrastructure/CICD_Pipeline/test_server_01/SleepApp-1.0-sleep-10' [type: cmd.DeployedCommand]
    }
  }
}
```
You can also redirect the output to a file and use the `apply` command, just as with previous 7.5.0 version of `xld-py-cli`:

```
(pyLightCliPy3) CristinasMBP:dev cristina$ xld --url http://sprintly.xebialabs.com:4516 --username yourXLDUsername --password yourXLDPass generate /Infrastructure/CICD_Pipeline > Deployfile
(pyLightCliPy3) CristinasMBP:dev cristina$ xld --url http://sprintly.xebialabs.com:4516 --username yourXLDUsername --password yourXLDPass apply DeployfileConcourseCI
```

### Other improvements

Besides the added Python 3.x support, the 7.6.0 release of `xld-py-cli` is based on the ["Fire Lite." version](https://github.com/google/python-fire/releases/tag/v0.1.2) of the popular library [Google Python Fire](https://github.com/google/python-fire) which makes it even more ["light weight"](https://github.com/google/python-fire/issues/7) pun intended.

On top of that:

* a new `deploy` command has been added that provisions an application version to an environment:
```
(pyLightCliPy3) CristinasMBP:dev cristina$ xld --url http://sprintly.xebialabs.com:4516 --username yourXLDUsername --password yourXLDPass deploy /Applications/PetClinic-ear/2.0 /Environments/Development/Dev
6913cbfc-a7ae-4b30-aff3-f8c1172bdfcf
```
The output is the newly created deployment task id and you will be able to see the result via the XLD GUI:
<img class="img-responsive" src="{{ site.baseurl }}/img/posts/xld-deploy-cli-python3/xld-py-cli-deploy.png" alt="XL Deploy GUI"/>
* a new `help` command is available that lists a simple usage of the XLD lightweight CLI

### Future perspective

Python 2.x interpreter compatibility is still warranted.

```bash
CristinasMBP:dev cristina$ virtualenv pyLightCliPy2
New python executable in /Users/cristina/dev/pyLightCliPy2/bin/python
Installing setuptools, pip, wheel...done.
CristinasMBP:dev cristina$ source pyLightCliPy2/bin/activate
(pyLightCliPy2) CristinasMBP:dev cristina$ pip freeze
(pyLightCliPy2) CristinasMBP:dev cristina$ pip install /Users/cristina/dev/xldeploy-py/dist/xldeploy-py-0.0.9.tar.gz
Processing ./xldeploy-py/dist/xldeploy-py-0.0.9.tar.gz
Collecting requests==2.11.1 (from xldeploy-py==0.0.9)
  Using cached requests-2.11.1-py2.py3-none-any.whl
Building wheels for collected packages: xldeploy-py
  Running setup.py bdist_wheel for xldeploy-py ... done
  Stored in directory: /Users/cristina/Library/Caches/pip/wheels/a9/5c/ae/9c9480a205f0030b5f49d91055834b912549021a349ceb2a02
Successfully built xldeploy-py
Installing collected packages: requests, xldeploy-py
Successfully installed requests-2.11.1 xldeploy-py-0.0.9
(pyLightCliPy2) CristinasMBP:dev cristina$ pip freeze
requests==2.11.1
xldeploy-py==0.0.9
(pyLightCliPy2) CristinasMBP:dev cristina$ pip install /Users/cristina/dev/xl-deploy/xld-py/xld-py-cli/build/distributions/xld-py-cli-7.6.0-SNAPSHOT.tar.gz
Processing ./xl-deploy/xld-py/xld-py-cli/build/distributions/xld-py-cli-7.6.0-SNAPSHOT.tar.gz
Requirement already satisfied: xldeploy-py in ./pyLightCliPy2/lib/python2.7/site-packages (from xld-py-cli===7.6.0-SNAPSHOT)
Collecting fire (from xld-py-cli===7.6.0-SNAPSHOT)
Requirement already satisfied: requests==2.11.1 in ./pyLightCliPy2/lib/python2.7/site-packages (from xldeploy-py->xld-py-cli===7.6.0-SNAPSHOT)
Collecting six (from fire->xld-py-cli===7.6.0-SNAPSHOT)
  Using cached six-1.11.0-py2.py3-none-any.whl
Building wheels for collected packages: xld-py-cli
  Running setup.py bdist_wheel for xld-py-cli ... done
  Stored in directory: /Users/cristina/Library/Caches/pip/wheels/cc/ff/cc/747a6d4e5e77d707eff8330b2f99c9bc77b5d3e59db0d67149
Successfully built xld-py-cli
Installing collected packages: six, fire, xld-py-cli
Successfully installed fire-0.1.2 six-1.11.0 xld-py-cli-7.6.0-SNAPSHOT
(pyLightCliPy2) CristinasMBP:dev cristina$ pip freeze
fire==0.1.2
requests==2.11.1
six==1.11.0
xld-py-cli===7.6.0-SNAPSHOT
xldeploy-py==0.0.9
(pyLightCliPy2) CristinasMBP:dev cristina$ xld version
xld-py-cli 7.6.0-SNAPSHOT from /Users/cristina/dev/pyLightCliPy2/lib/python2.7/site-packages/xld (python 2.7.10)
```

And of course hereby some useful [pointers](http://python-future.org/compatible_idioms.html) on writing Python 2-3 compatible code. Thanks for reading and enjoy command-line interfaces!
