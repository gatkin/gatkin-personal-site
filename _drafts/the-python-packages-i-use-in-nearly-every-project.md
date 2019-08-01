---
layout: post
title: The Python Packages I Use in Nearly Every Project
subtitle: ''
date: 2018-11-25 06:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: There are a handful of Python packages that I have found to be very useful
  in the Python projects I have worked on which I expect to be useful to a significant
  portion of Python projects in general

---
There are a handful of Python packages that I have found to be very useful in the Python projects I have worked on which I expect to be useful to a significant portion of Python projects in general. These projects cover a few common development concerns faced by many Python projects such as testing, static analysis, dependency management, and utility libraries.

- Testing
    - pytest
    - coverage.py
- Static Analysis
    - flake8
    - pylint
    - mypy
- Dependency Management
    - pipenv
- Utility Libraries
    - attrs

These are just some of the libraries I have found useful in my first few years of working in Python. I am sure I will discover other new and useful packages as I continue to grow and learn as a Python developer.

## Testing

My experience with testing in Python projects has been an absolute joy, largely due to the quality of testing libraries available in Python.

### pytest

[Pytest](https://docs.pytest.org/en/latest/) is hands-down my favorite testing framework I have every worked with in any language. I particularly love the simplicity of the assertion introspection feature it provides. All you need is the `assert` keyword and pytest handles generating the output necessary for diagnosing any test failures no matter the complexity of the values being compared in the assert statement.

{% gist 612f95668b3ab93b6dc5389c09e479a8 %}

{% gist 1a42a2a15b5f5a2b221ef53ddd419dda %}

In addition, the auto-discovery and test fixture features provided by pytest help greatly in writing high-quality, maintainable tests.

### coverage.py

For measuring code coverage in Python, [coverage.py](https://coverage.readthedocs.io/en/latest/) is my go-to solution. It is simple to integrate with pytest tests and produces nice coverage reports in both terminal and HTML formats. coverage.py can also measure branch coverage and allows you to fail builds that fall under a minimum coverage treshold.

### Other Testing Packages

- [cosmic-ray](https://cosmic-ray.readthedocs.io/en/latest/index.html) - A mutation testing library that provides a good indication of the effectiveness of a test suite. I have used this in only a few Python projects.
- [hypothesis](https://hypothesis.readthedocs.io/en/latest/) - A property-based testing library that I have not used before but would like to add to my projects.
- [tox](https://tox.readthedocs.io/en/latest/) - A tool for testing Python packages across multiple Python environments that I would like to explore integrating into the CI testing for my Python projects.

## Static Analysis

In a dynamic language like Python, static analysis tools are tremendously helpful in catching issues early and maintaining code quality. I always run these static analysis tools in my continuous integration builds.

### flake8

[flake8](http://flake8.pycqa.org/en/latest/index.html) is a linter and style-checker for Python code. It helps ensure consistent code style and can catch some basic coding errors.

### pylint

[pylint](https://www.pylint.org/) is a fairly thorough linter that can catch even more errors than flake8. pylint can sometimes be too strict and flag constructs in code that we do not view as issues to be fixed. When adding pylint to a project, I find it most effective to create a baseline pylint configuration file by running `python -m pylint --generate-rcfile` which includes a long list of warnings that I generally like to ignore.

I usually use both flake8 and pylint together in projects since they flag non-overlapping sets of code errors.

### mypy

[Mypy](http://mypy-lang.org/) is a static type checker for Python code. It uses type annotations as specified in [PEP 484](https://www.python.org/dev/peps/pep-0484/) to perform its type checking. Mypy supports *gradual typing* meaning types can be added slowly to an existing code base. Both Python 3 and Python 2.7 are supported

{% gist f5a98e4e28dae3d8bc865368dc4236de %}

{% gist 07dc29e6cf40919d1fcf36c51f66a73f %}

While adding static types to my Python code has caught a few errors, I find that static typing provides the most value as a form of machine-checked documentation that you know is always up to date. Since mypy supports, gradual typing, you do not have to annotate everything in the project to gain a lot of value. I find  that adding type annotations around the core data types and key functions in my project goes a long way towards improving understandability and maintainability particularly as the project grows in size.

Facebook has also open sourced a static type checker for Python called [Pyre](https://pyre-check.org/) which I have not used before but looks to support many of the same features as mypy.

## Dependency Management

### pipenv

[Pipenv](https://pipenv.readthedocs.io/en/latest/) is my go-to package manager tool because it manages both installing packages and virtual environments in addition to providing a great way to specify and manage project dependencies. It solves many of the pain points I encountered with using `pip`, `virtualenv`, and `requirements.txt` files individually.

Besides its transparent management of virtual environments, my favorite feature of `pipenv` is how well it allows you to specify project dependencies. With `pipenv`, you specify only the *top-level* dependencies in a `Pipfile` distinguishing between regular dependencies and development-only dependencies. Unlike a `requirements.txt` file, a `Pipfile` does not contain any transitive dependencies. This allows you to quickly see exactly the packages your project depends on directly.

{% gist bac528ca926c88e1dd153fe5c4be4e13 %}

A `Pipfile.lock` file is generated to pin the exact versions of all top-level and transitive dependencies.

### Other Dependency Management Tools

- [Poetry](https://poetry.eustace.io/) - Although I have not worked with it yet, the Poetry tool seems very intriguing because it appears to provide many of the features of `pipenv` while also making assembling and publishing Python packages much simpler.

## Utility Libraries

### attrs

The [attrs](https://www.attrs.org/en/stable/) is the most useful Python package I have come across so far, and I wish I had known about it when I was just starting working with Python. It enables you to quickly and easily write classes that represent simple, dumb data values with all the dunder methods (`__repr__`, `__eq__`, `__hash__` etc) generated for you

{% gist dc3c55afb4227ea491b61058b46a4b10 %}

The `attrs` library is particularly useful in enabling a more functional, data-oriented style of development because it makes it very easy to define new value types and even allows you to make those types immutable. This creates a similar experience to working in a more functional language such as F#.