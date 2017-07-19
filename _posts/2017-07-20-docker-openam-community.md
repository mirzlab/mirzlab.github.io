---
layout: post
title: Docker image for OpenAM community version
published: false
---

Here is my OpenAM Docker image, made to quickly have an up and ready OpenAM 11.0.3 server.

**This Docker image is good for quick tests or POCs, not intended for production use.**

-----

```bash
git clone git@github.com:mirzlab/docker-openam-community.git
cd docker-openam-community
```

# How to use

## Quick run

```sh
$ docker build . -t openam
$ docker run -it --rm --add-host "openam.example.com:127.0.0.1" -p 8443:8443 openam
```

The OpenAM server will be available at https://openam.example.com:8443/openam.

Update your /etc/hosts file on your host machine if necessary.

## Customization

Some options are available to customize the OpenAM server.

### Default build parameters

| Parameter             | Description                                       | Default value      |
|-----------------------|---------------------------------------------------|--------------------|
| OPENAM_HTTPS          | If you need the OpenAM webapp to run using https  | true               |
| TOMCAT_HTTPS_PORT     | Tomcat https port                                 | 8443               |
| TOMCAT_HTTP_PORT      | Tomcat http port                                  | 8080               |
| OPENAM_HOST           | OpenAM server host                                | openam.example.com |
| OPENAM_DEPLOYMENT_URI | OpenAM server uri                                 | openam             |
| OPENAM_ADMIN_PASSWORD | OpenAM amadmin password                           | Admin001           |

### Custom configuration

Here some some examples of custom configuration:

```bash
$ docker build . -t openam \
--build-arg OPENAM_HOST=demo.openam.com \
--build-arg OPENAM_DEPLOYMENT_URI=sso \
--build-arg OPEANM_ADMIN_PASSWORD=P@ssw0rd \
--build-arg OPENAM_HTTPS=false \
--build-arg TOMCAT_HTTP_PORT=8888
$ docker run -it --rm --add-host "demo.openam.com:127.0.0.1" -p 8888:8888 openam
```

```bash
$ docker build . -t openam \
--build-arg OPENAM_HOST=demo.openam.com \
--build-arg OPENAM_DEPLOYMENT_URI=sso \
--build-arg OPEANM_ADMIN_PASSWORD=P@ssw0rd01 \
--build-arg OPENAM_HTTPS=true \
--build-arg TOMCAT_HTTPS_PORT=8443
$ docker run -it --rm --add-host "demo.openam.com:127.0.0.1" -p 8443:8443 openam
```

Tomcat HTTPS connector is configured using a generated self signed certificate.

