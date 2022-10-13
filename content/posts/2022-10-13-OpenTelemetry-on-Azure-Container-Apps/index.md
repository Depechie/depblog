---
title: "OpenTelemetry on Azure Container Apps"
date: 2022-10-13T21:48:17Z
categories: [development,observability]
tags: [opentelemetry,azure,containerapps]
draft: false
---

Lately I've been working with `OpenTelemetry` to setup an observability environment for our running api services. Currently everything is deployed on-premises through use of `docker compose` and is working perfectly, but of course we also wanted to try out a cloud hosted solution on `Azure`.

So as an experiment I decided to try out `Azure Container Apps` to see how it would work with `OpenTelemetry`. But truth be told the experience was not as smooth as I had hoped for. The Microsoft documentation is very basic when it comes to what kind of containers you can setup with `Azure Container Apps` and it's not very clear what kind of limitations there are.

Thankfully [Martin Thwaites](https://twitter.com/MartinDotNet), a Developer Advocate at [@honeycombio](https://twitter.com/honeycombio) and also an `OpenTelemetry` enthouisast, had already done some work on this and had a [blog post](https://www.honeycomb.io/blog/opentelemetry-collector-azure-container-apps) on the subject. So I decided to follow his guide and see if I could get it to work.

Have to say, without this guide, I would have been lost. So thanks Martin!

> **Note** Seeing what Martin has done with his blogpost and then noting what I still had to tweak manually, I would think **Microsoft** still has some work to do to make this experience for hosting docker container better.

So why still this blogpost? Well even though the original blog post has a very detailed guide, I still had troubles on making it work for my particular setup. So I decided to write this blogpost to help others that might be in the same situation as me.

There are 2 things that I had to change in reference to the original blog post, so that my use case would actually work.

#### OpenTelemetry Collector Configuration file

Could be this part is not actually needed, but the setup from Martin somehow did not do the trick and since I'm used to using a command line command with and argument defined in the docker comopse file to indicate what configuration file `OpenTelemetry` needs at startup, I needed a way to replicate this setup in with `Azure Container Apps`.

What Martin already pointed out in his blogpost, is the fact that we need to download the container app configuration once it has been created to edit this manually for hooking up the file container mounts. Details are here [https://www.honeycomb.io/blog/opentelemetry-collector-azure-container-apps#step__add_the_container](https://www.honeycomb.io/blog/opentelemetry-collector-azure-container-apps#step__add_the_container).

It's this section that I had to further change to reflex the same setup as the one in my docker compose file.
So my working configuration file looks like this:

```
  template:
    containers:
    - image: otel/opentelemetry-collector-contrib:0.61.0
      name: acaoteltestdev
      args:
        - "--config=/etc/otel-config-test.yml"
      resources:
        cpu: 0.5
        ephemeralStorage: ''
        memory: 1Gi
      volumeMounts:
      - mountPath: /etc
        volumeName: config
    revisionSuffix: ''
    scale:
      maxReplicas: 1
      minReplicas: 1
    volumes:
    - name: config
      storageName: stsmountoteltestdev
      storageType: AzureFile
```

> **Note** The `args` section is the one that I had to add to make it work and with that I also had to change the `volumeMounts` section to reflect the same path.

#### Grafana Cloud

The other deviation form the original blog post is the fact that I'm using `Grafana Cloud` instead of `Honeycomb`. So I had to change the `OpenTelemetry` configuration file to reflect this.

But this also was not without some setbacks, because the logs from the `OpenTelemetry collector` container were not always showing an exception, even though I did not receive any data in `Grafana Cloud`.

In the end it was a combination of setting up the correct `Basic Authentication` header and skipping the `TLS` verification. So my working configuration file looks like this:

```
receivers:
  otlp:
    protocols:
      http:

exporters:
    otlp:
      endpoint: <Your Grafana Cloud endpoint>:443
      headers:
        authorization: Basic <base64 representation of your Grafana Cloud key>
      tls:
        insecure: false
        insecure_skip_verify: true

service:
    pipelines:
        traces:
        receivers: [otlp]
        exporters: [otlp]
```

And with that all in place everything started working as expected!
Hopefully this post together with the original blog post from Martin will help others that are trying to get `OpenTelemetry` working with `Azure Container Apps`.