---
layout: post
title: Deploying Dockerized ASP .NET Core Applications to Heroku
subtitle: ''
date: 2019-08-22 06:00:00 +0000
thumb_img_path: ''
content_img_path: ''
excerpt: 'In this post, I describe how to run Dockerized ASP.NET Core applications on Heroku.'

---

Heroku is a great, low-cost option for hosting small hobby projects. Unfortunately, the .NET Core platform is not an officially supported runtime on Heroku. However, you can still use Heroku as a hosting solution for ASP .NET Core applications with [Heroku's Container Runtime](https://devcenter.heroku.com/articles/container-registry-and-runtime). In this post, I describe how to run Dockerized ASP .NET Core applications on Heroku.

A repository that shows an example ASP .NET Core application that is deployed to Heroku can be found at [https://github.com/gatkin/heroku-net-core-example](https://github.com/gatkin/heroku-net-core-example).


## Creating the Dockerfile

Once you have an ASP.NET Core application you would like to deploy to Heroku, you will need to create a `Dockerfile` similar to the following

{% gist 1a112d998ecbf1b6a5f918e7bf0b5f12 %}

There are a couple of things to note with this `Dockerfile` setup

#### Use `CMD` Instead of `ENTRYPOINT`

If you use `ENTRYPOINT` in your `Dockerfile`, you may get [an error](https://stackoverflow.com/q/55913408/4517653) when deploying the image to Heroku of

```log
No command specified for process type web
```

Using `CMD` instead of `ENTRYPOINT` will fix this error.

#### Run as a Non-Root User Inside the Container

Because containers [are not run with root privileges on Heroku](https://devcenter.heroku.com/articles/container-registry-and-runtime#run-the-image-as-a-non-root-user), you should run your application as a non-root user inside the container. Otherwise, you might see errors when attempting to start up your application.

#### Listen on the Correct Port

The Heroku container runtime provides the port on which a container application can listen on for incoming HTTP requests through the [`$PORT` environment variable](https://devcenter.heroku.com/articles/container-registry-and-runtime#dockerfile-commands-and-runtime). Therefore, you must provide that port variable to the command to start your ASP .NET core application.


## Enabling HTTPS Redirects

Due to how Heroku routing works, the [`UseHttpsRedirection` middleware](https://docs.microsoft.com/en-us/aspnet/core/security/enforcing-ssl?view=aspnetcore-2.2&tabs=visual-studio#usehttpsredirection) does not work when running an ASP .NET core application on Heroku. The only way to tell whether an incoming request is an HTTP or an HTTPS request is by checking the `x-forwarded-proto` header as described in [this blog post](https://jaketrent.com/post/https-redirect-node-heroku/).

To enable HTTPS redirects on Heroku, you can add the following middleware component to your application

{% gist c7e9a8ca8428646d76992b224d1dfa3e %}

To use this middleware component, add the following to the `Configure` method in the `Startup` class of your ASP .NET application

{% gist 992134cc0410e45e0fb9b029abc56d29 %}