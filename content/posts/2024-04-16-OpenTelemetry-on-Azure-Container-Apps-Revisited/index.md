---
title: "OpenTelemetry on Azure Container Apps Revisited"
date: 2024-04-15T18:43:59Z
categories: [development,observability]
tags: [opentelemetry,azure,containerapps]
draft: true
---

https://rimazmohommed523.medium.com/containerize-a-net-core-web-api-app-with-docker-2cdba52a8978

On the console, inside the src folder, run the `docker init` command to create a new Dockerfile.

I added the

```
EXPOSE 5209
ENV ASPNETCORE_URLS=http://+:5209
```

**TODO** Check if this is needed! The docker init asks for a port number!

Removing the docker build cache, run command `docker buildx prune -f`

To try out the image pushed to your own docker registry

https://learn.microsoft.com/en-us/azure/container-registry/container-registry-get-started-docker-cli?tabs=azure-cli

To test pushed images to ACR locally, run the following commands:
```
az login
az acr login --name crdevwesteuacaotel
docker pull crdevwesteuacaotel.azurecr.io/apiservice:latest
```

For the ACA build pipeline yaml file, we need to set a variable called azureContainerRegistryPassword in devops! Go to the edit pipeline screen and click on the variables tab.

The password can be found on the Container Registry in the Access keys section, the bicep file has the correct user name set. It will be the same as the registry name per default.