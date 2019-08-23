---
layout: post
title: Automating PyPI Deployments with Azure Pipelines
subtitle: ''
date: 2019-08-03 06:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: Making releasing to PyPI as simple as creating a new git tag.

---

Publishing Python packages to PyPI can sometimes be difficult. After publishing a few of my own Python packages to PyPI, I have found a setup for automating the process of releasing to PyPI using Azure Pipelines that makes publishing a package to PyPI as simple as creating a new git tag.

While this walkthrough uses Azure Pipelines, this setup can be modified to work on your CI provider of choice.

An example repository which implements this setup can be found at [https://github.com/gatkin/automated-pypi-deploy-example](https://github.com/gatkin/automated-pypi-deploy-example).

## Project Setup

To start with, ensure your project is configured with a valid `setup.py` file so that it can be packaged into a distributable format as described in the [Python Packaging User Guide](https://packaging.python.org/tutorials/packaging-projects/).

## PyPI Account Setup

If you do not have one already, you will need to [create a PyPI account](https://pypi.org/account/register/) that you will use to upload your package.

## Build Scripts

I often put all of my build scripts in a Makefile for my projects. I find this offers a few advantages over putting build scripts directly in the CI YAML configuration file:

- It is easier to reduce code duplication between different build tasks
- The build tasks can be run locally the same way they are run on the CI server to enable easier local testing and diagnosis of CI failures

A minimal Makefile that I use for Python projects I publish to PyPI is

{% gist 9ba1836475c29e6de51895fe5baceadb %}

[Twine](https://twine.readthedocs.io/en/latest/) is used to perform most of the heavy lifting of publishing a package to PyPI. The PyPI credentials are provided via environment variables from the CI server to keep authentication information out of source control and secret.

## Azure Pipeline Setup

### Add PyPI Credentials as Secret Environment Variables

Add your PyPI username and password as secret environment variables for Azure Pipeline builds so that your credential information can stay out of source control and CI build logs and remain secret.

To do so, first select the "Edit" button on your pipeline

![](/images/automating-pypi-deployments/EditPipeline.png)

Then, select the "Variables" button to add new environment variables

![](/images/automating-pypi-deployments/EditVariables.png)

Finally, add your PyPI username and password as secret environment variables under `PYPI_USERNAME` and `PYPI_PASSWORD`

![](/images/automating-pypi-deployments/AddVariable.png)


### Create an `azure-pipelines.yml` File

The final step is to create an `azure-pipelines.yml` file at the root directory of your repository. A minimal `azure-pipelines.yml` that I use for Python projects I publish to PyPI is

{% gist cb14958207b90a893447810d925ae09b %}

This configures Azure Pipelines to trigger builds on every commit to all branches and on any new tags that start with a `release/` prefix. The `Build` job runs on every build on every branch. In a real project, this job would perform tasks such as running linters, running unit tests, and reporting on code coverage.

The `Release` job does all the work of publishing the package to PyPI. It only runs after the `Build` job completes successful and when the pipeline has been triggered by a new tag that starts with `releases/`.

## Triggering a Release to PyPI

Now, all you need to do to publish a new version of your package to PyPI is to update the `setup.py` file to specify the new version of your package and create a new tag that starts with `release/`. Creating the new tag will automatically trigger an Azure Pipeline job which will build and publish the new version of your package to PyPI.

## Possible Improvements

One major drawback of this setup is that you have to always be sure to update the `setup.py` file with the new version of your package before creating the tag to trigger a new release. A nice improvement would be to have the package version specified through the tag name. For example, creating a tag named `releases/1.2.0` would publish the package with version `1.2.0`.
