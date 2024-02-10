---
title: 'Python project setup best practices'
date: 2024-02-10 12:00:00
categories: [software-development]
tags: [python]
toc: true
---

## Best practice
* have a version of Python for your project independent of your system and global Python
  - we can achieve this using pyenv

* deterministically manage the dependencies on a project basis
   - we can achieve this with Poetry  

The biggest anti-pattern is using a global Python with globally installed packages. It's just un deployable and very hard to replicate. 

## Setup

Install pyenv and poetry on your machine. On Mac use: `brew install pyenv poetry`

To make sure the `.venv` created by Poetry is in the same directory of your project, add a `poetry.toml` with following contents:
```
[virtualenvs]
in-project = true
```

## pyenv

You should decide on a specific version of Python for your project.

If you want to use Python 3.11.8 instead of the latest Python 3.12, in your project your should run: `pyenv local 3.11.8` (feel free to adapt the guide to any version). 

This should create a file: `.python-version` containing the version.

NOTE: You might need to update pyenv first with `pyenv update` and then run `pyenv install <version>` (like 3.11.8) if you don' have the desired version locally.

## Poetry

Use Poetry to define your projects dependencies. 

Poetry, within it's `pyproject.toml`, recommends to define the version of Python, for example you could do have this:
```
[tool.poetry.dependencies]
python = "~3.11"
... continue with other deps ...
```

Then get poetry to use the version of Python you installed with pyenv: `poetry env use 3.11.8`

## Check the setup 

Just running `poetry env info` should confirm we are using a pyenv install Python and a .venv folder installed within our project. Example output:

```
➜ poetry env info
                       
Virtualenv
Python:         3.11.8
Implementation: CPython
Path:           <project>/.venv
Executable:     <project>/.venv/bin/python
Valid:          True

System
Platform:   darwin
OS:         posix
Python:     3.11.8
Path:       $HOME/.pyenv/versions/3.11.8
Executable: $HOME/.pyenv/versions/3.11.8/bin/python3.11
```

This simplicity of setup wasn't in Python five years ago. So it's a pleasure to see how the Python ecosystem has matured.  
