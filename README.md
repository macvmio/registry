# Registry Setup Guide

This guide provides instructions for setting up a **self-signed Docker registry** that acts as an **OCI artifacts registry**, serving both **container images** and **Curie VM images** through **Geranos**. The registry supports deduplication of blobs, making it an efficient and secure storage solution for similar VM images. This guide focuses on setting up a Docker V2 registry, optimized for storing Curie VM images using Geranos.

For production environments, we recommend using **Minio** for on-premises S3-compatible storage to ensure scalability and durability. Additionally, you can integrate the `cosign` tool to sign your VM images, providing a higher level of security and preventing image tampering.

## Table of Contents
- [Overview](#overview)
- [Public and Private Registries](#public-and-private-registries)
- [Setting up an OCI-Compatible Docker V2 Registry](#setting-up-an-oci-compatible-docker-v2-registry)
  - [1. Registry Configuration](#1-registry-configuration)
  - [2. Auth Configuration](#2-auth-configuration)
  - [3. Running the Registry](#3-running-the-registry)
- [Efficient Storage with Blob Deduplication](#efficient-storage-with-blob-deduplication)
- [Security with Cosign](#security-with-cosign)
- [GoHarbor Integration](#goharbor-integration)
- [Useful Links](#useful-links)

## Overview

This registry setup focuses on **Docker V2**, but it is fully compliant with **OCI artifacts**, making it capable of storing not only traditional container images but also **Curie VM images** through the **Geranos** tool. The ability to handle both containers and VM images ensures a unified storage solution that works seamlessly across different types of workloads.

In production, we recommend backing the registry with **Minio** for S3-compatible storage, which ensures scalability and robustness. The registry also efficiently **deduplicates blobs**, reducing the storage footprint while maintaining high performance for storing multiple versions of similar VM images.

## Public and Private Registries

### Public Registries
- **Docker Hub**: The default public registry used by Docker.
- **GitHub Container Registry (GHCR.io)**: Public and private OCI-compatible registry hosted by GitHub.

### Private Registries
- **GoHarbor**: An open-source Docker V2 and OCI-compatible registry solution offering advanced features like RBAC, image signing, and vulnerability scanning.
- **AWS ECR**: Amazon's Elastic Container Registry, offering high scalability.
- **Google Cloud Container Registry (GCR)**: Google's fully managed OCI-compatible registry.

## Setting up an OCI-Compatible Docker V2 Registry

### 1. Registry Configuration

Below is a sample configuration for setting up a **Docker V2 registry** that serves as an OCI artifacts registry. This configuration is designed to store both container images and VM images, utilizing local filesystem storage in this example. However, for production setups, it is recommended to use **Minio** for S3-compatible storage.

#### `/etc/default/registry.yaml`:
```yaml
version: 0.1
log:
  fields:
    service: registry
auth:
  token:
    realm: "https://macvm.store/auth"
    service: "jarosik-oci-service"   # Must match service in auth_config.yml
    issuer: "jarosik-token-issuer"  # Must match issuer in auth_config.yml
    rootcertbundle: /etc/default/registry-certs/cert.pem # Public cert in Docker v2.8
storage:
  delete:
    enabled: true
  filesystem:
    rootdirectory: /mnt/ssd850a/registry-data
  # For production, configure S3-compatible storage (Minio) for durability and scalability.
  # s3:
  #   accesskey: <YOUR-ACCESS-KEY>
  #   secretkey: <YOUR-SECRET-KEY>
  #   region: "us-east-1"
  #   bucket: goharbor-vm-images
  #   endpoint: "http://minio.example.com"
  cache:
    blobdescriptor: inmemory
http:
  addr: :5000
  secret: thisisasecretforhttp
  host: "https://macvm.store"
```

To learn more about using **Minio** as your on-premises S3-compatible storage solution, visit the [Minio documentation](https://min.io/docs/minio/linux/index.html).

### 2. Auth Configuration

We will use the **Docker Auth** tool from Cesanta to manage access to your registry. This tool provides a flexible way to authenticate users using a variety of methods, such as token-based authentication or static user maps.

You can find more details about **Docker Auth** at its official repository: [Cesanta Docker Auth](https://github.com/cesanta/docker_auth).

#### `/etc/default/registry-auth.yml`:
```yaml
server:
  addr: ":5001"

token:
  issuer: "jarosik-token-issuer"  # Must match issuer in registry config.
  expiration: 900
  certificate: "/etc/default/registry-certs/cert.pem"
  key: "/etc/default/registry-certs/privkey.pem"

users:
  # BCrypt-hashed passwords for users
  "tjarosik":
    password: "<redacted>"
  "marcin":
    password: "<redacted>"
  "test":
    password: "<redacted>"
  "": {}  # Anonymous (no login) access

acl:
  - match: {account: "tjarosik"}
    actions: ["*"]
    comment: "Allow all actions for user"
  - match: {account: "marcin"}
    actions: ["*"]
    comment: "Allow all actions for user"
  - match: {account: "", name: "public/*"}
    actions: ["pull"]
    comment: "Anonymous users can pull from public/* images"
```

### 3. Running the Registry

Once the configuration files are ready, you can start the registry and authentication server using Docker. Below are the commands:

#### Starting the registry:
```bash
docker run -d --name registry \
  -p 5000:5000 \
  -v /etc/default/registry.yaml:/etc/docker/registry/config.yml \
  -v /mnt/ssd850a/registry-data:/var/lib/registry \
  registry:2
```

#### Starting the authentication server:
```bash
docker run -d --name registry-auth \
  -p 5001:5001 \
  -v /etc/default/registry-auth.yml:/auth/config.yml \
  cesanta/docker_auth:latest
```

## Efficient Storage with Blob Deduplication

Docker V2 registry, as used with Geranos, supports **blob deduplication**, which is crucial when dealing with VM images. When multiple VM images share common layers or blobs, the registry will automatically deduplicate them, storing only one copy of each blob. This feature optimizes storage usage, making it efficient to store various versions of similar VM images without redundant data.

For even greater efficiency in production, configure the registry to use **Minio** as your on-premises S3-compatible storage solution. This ensures high availability, durability, and performance in large-scale environments.

## Security with Cosign

To ensure the integrity of your VM images, we recommend using the `cosign` tool, which allows you to sign your VM images. By signing images, you can prevent tampering and ensure that only verified images are pulled and deployed. Cosign integrates seamlessly with Docker V2 registries, providing an extra layer of security for your VM images.

Here’s an example of how to sign an image with `cosign`:
```bash
cosign sign ghcr.io/macvmio/vm-image:macos-15.0.1-base
```

For more information on `cosign`, visit the [Cosign GitHub repository](https://github.com/sigstore/cosign).

## GoHarbor Integration

For those seeking more advanced features, such as RBAC, content signing, and vulnerability scanning, consider using **GoHarbor**. While this guide focuses on a basic Docker V2 setup, GoHarbor offers an enterprise-grade registry solution that integrates well with Kubernetes and cloud environments.

Refer to the [GoHarbor documentation](https://goharbor.io/docs) for setup instructions, and check the [GoHarbor Releases](https://github.com/goharbor/harbor/releases) for the latest updates.

## Useful Links

- **Docker V2 Registry Releases**: Stay updated with the latest releases at [Docker Registry Releases](https://github.com/distribution/distribution/releases).
- **GoHarbor Releases**: Download and check the latest features of GoHarbor at [GoHarbor Releases](https://github.com/goharbor/harbor/releases).
- **Minio Documentation**: Learn more about Minio for on-premises S3 storage [here](https://min.io/docs/minio/linux/index.html).
- **Cosign**: Learn more about signing your VM images with Cosign [here](https://github.com/sigstore/cosign).
