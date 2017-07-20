---
layout: post
title: Docker image for OpenAM community version
---

Here is my OpenAM Docker image, made to quickly have an up and ready OpenAM 11.0.3 server.

**This Docker image is made for quick tests or POCs, not intended for production use.**

-----

```bash
git clone git@github.com:mirzlab/docker-openam-community.git
cd docker-openam-community
```

[Github link](https://github.com/mirzlab/docker-openam-community)

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

<pre>
<b>OPENAM_HTTPS</b>
If you need the OpenAM webapp to run using https

Default value: <b>true</b>
</pre>

<pre>
<b>TOMCAT_HTTPS_PORT</b>
Tomcat https port

Default value: <b>8443</b>
</pre>

<pre>
<b>TOMCAT_HTTP_PORT</b>
Tomcat http port

Default value: <b>8080</b>
</pre>

<pre>
<b>OPENAM_HOST</b>
OpenAM webapp will be configured using this host

Default value: <b>openam.example.com</b>
</pre>

<pre>
<b>OPENAM_DEPLOYMENT_URI</b>
OpenAM server uri (i.e < host ><b>/openam</b>

Default value: <b>openam</b>
</pre>

<pre>
<b>OPENAM_ADMIN_PASSWORD</b>
OpenAM amadmin (admin user) password

Default value: <b>Admin001</b>
</pre>

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

