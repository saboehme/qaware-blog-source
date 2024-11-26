---
title: "Crafting a Good Java Container Image for Cloud-Native Applications"
date: 2024-11-26T11:27:00+01:00
lastmod: 2024-11-26T11:27:00+01:00
author: "[Sascha Böhme](https://github.com/saboehme)"
type: "post"
image: "java-container-image.png"
tags: ["Java", "Cloud-native", "Container", "Distroless", "Vulnerabilities", "Trivy"]
draft: true
summary: "Design decisions and strategies for Java container images with a focus on security and size." 
---

Small, reduced container images are a key factor for cloud-native applications.
Small container images contain only the necessary components to run the application, thereby reducing the attack surface.
Small container images are also faster to build, transfer and deploy.
Building a small container image for Java applications can be challenging if standard solutions are insufficient.

# Java Container Images - The Easy Way

In most cases, the existing Java container images are applicable and sufficient for cloud-native Java applications.
Some available images are currently (in alphabetical order):

* [Amazon Corretto](https://hub.docker.com/_/amazoncorretto)
* [Eclipse Temuring](https://hub.docker.com/_/eclipse-temurin)
* [Google Distroless](https://github.com/GoogleContainerTools/distroless)

There are many other alternatives to take.
Finding the right image is a matter of requirements and constraints, and the existing images are often sufficient
for standard use cases.

# Reasons for Custom Java Container Images

When it comes to security, a bloated image that contains many, mostly unused tools can be a security risk.
The more tools are available, the more potential vulnerabilities are present.
OpenSSL, for example, is a tool commonly found in Java container images.
Vulnerabilities in OpenSSL are discovered regularly. Yet, Java applications do not require OpenSSL in general.
Java applications use the Java Secure Socket Extension (JSSE) for secure communication.
OpenSSL is only required if the Java application invokes native code that builds on OpenSSL, and that is a rare case.
Similar to OpenSSL, there are many more tools in standard Java container images that are likely to be unused
in most Java applications.

Albeit not being used in most cases, the artifacts in the container image are a potential security risk.
Analysis tools like [Trivy](https://trivy.dev/) can help to identify vulnerabilities in container images.
They gain more importance as cloud providers increasingly scan container images for vulnerabilities.
AWS, for instance, automatically scans container images for vulnerabilities with [Amazon ECR](https://aws.amazon.com/ecr/).
Such tools do not differentiate between used and unused artifacts in the container image.
Any potential vulnerability in the image is a risk.

Besides security, the size of the container image is another reason for custom Java container images.
The smaller the image, the faster it can be built, transferred and deployed.
Smaller images thus reduce costs and improve the overall performance of the application.

There are further reasons for custom Java container images, such as:

* A specific Java distribution is required, for instance due to support or licensing reasons.
* Additional system libraries are required, for instance if Java libraries that rely on native code are applied.
* Additional tools are required, for instance for monitoring or debugging purposes.

If such requirements are not met by the existing images, the standard approach is to create a custom image.
This is where the challenges begin.

# Java Container Images - The Hard Way

Building a custom Java container image seems to be an easy thing on first sight.
In fact, the following Dockerfile is sufficient to create a Java container image:

```docker
ARG ZULU_VERSION=11.0.12
FROM azul/zulu-openjdk-debian:$ZULU_VERSION as builder

FROM gcr.io/distroless/base-nossl-debian12

ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en' LC_ALL='en_US.UTF-8'

ENV JAVA_HOME=/usr/lib/jvm/zulu11
ENV PATH=${PATH}:${JAVA_HOME}/bin

COPY --from=builder $JAVA_HOME $JAVA_HOME
COPY --from=builder /lib/x86_64-linux-gnu/libz.* /lib/x86_64-linux-gnu/

COPY --from=amd64/busybox:1.31.1 /bin/busybox /busybox/busybox
RUN ["/busybox/busybox", "--install", "/bin"]

CMD ["/bin/sh", "-c", "java -version"]
```

A container image built this way contains the Zulu OpenJDK and a BusyBox shell for small administrative tasks and potentially runtime debugging.
The image contains only the necessary components to run a Java application.
It is small with a size of around ??? MB and has only minor violations (as of writing).
This image is a good starting point for custom Java container images, although some security hardening still needs to be applied.

## The Issue with Custom Java Container Images

Tools to identify security vulnerabilities in container images, such as Trivy, are currently limited.
Instead of scanning the entire filesystem, they typically look for traces of common package managers to identify installed packages and their versions.
Such an approach works well if the container image is based on a standard Linux distribution such as Debian, Alpine or RedHat.
The approach fails for custom Java container images that are put together from components of several other container images as in the example show before.
On that image, Trivy, for instance, does identify the following installed components:

* base-files
* libc6
* netbase
* tzdata

These are exactly the Debian packages in Google's Distroless image.
The Zulu OpenJDK, BusyBox, and zlib are missing from this list.
A security vulnerability in one of these three components will therefore never be detected by Trivy.
This is an untenable situation for a container image that is primarily used to run Java and in which Java makes up the majority of the size.

## Better Custom Java Container Images

To improve the situation, the custom Java container image must be built in a way that allows analysis tools such as Trivy to identify all installed components.
Simply copying artifacts from unrelated container images is hence not sufficient.

This observation rules out the use of Google's Distroless image as a base for a custom Java container image since it does not support modifications by construction.
With adequate knowledge of Bazel, the system that builds the Google Distroless images, a cloned and adapted container image might be possible.
Instead, the custom Java container image should be built from a base image that is based on a standard Linux distribution.
Canonical offers [chisel](https://github.com/canonical/chisel) as tool to carve out reduced images from Ubuntu distributions.
See also the [Ubuntu blog](https://ubuntu.com/blog/craft-custom-chiselled-ubuntu-distroless) for an example.
This tool only works with Ubuntu packages, though, and can hence not be applied to, for instance, the Zulu OpenJDK.
[Alpine](https://alpinelinux.org/) is a common choice for custom container images due to its small size and security focus.
Its usage of muslc, however, makes it somewhat esoteric and not suitable for all Java applications, especially those that rely on native code which is typically linked against glibc.
In the [Quarkus](https://quarkus.io/) environment, the [RedHat Universal Base Image (UBI)](https://catalog.redhat.com/software/base-images) is used by default.
This RedHat-based distribution fulfills major requirements for creating reduced Java container images such as customizable through a package manager, reduced to the necessary components, production-ready, and good support and maintenance.

The following Dockerfile shows how a custom Java container image can be built based on the RedHat UBI:

```docker
FROM busybox:stable-glibc AS busybox

FROM redhat/ubi9:latest AS builder

ARG DNF_FLAGS="--releasever 9 --setopt install_weak_deps=false --nodocs -y"

# This root folder will hold all artifacts that make up the final image.
# JDK and required libraries will be copied here.
RUN mkdir -p /mnt/rootfs

# Install the base system, required libraries and Java into the new root folder.
# Note that Java requires only glibc and zlib.
# The C++ libraries are added to support Java library that interface via JNI with native libraries built in C++.
# UBI includes further artifacts by default, such as bash, ncurses, and tzdata, that add little risk to violations.
RUN yum install -y https://cdn.azul.com/zulu/bin/zulu-repo-1.0.0-1.noarch.rpm && \
    dnf install --installroot /mnt/rootfs redhat-release ${DNF_FLAGS} --nogpgcheck &&  \
    rpm --root=/mnt/rootfs --import /mnt/rootfs/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release && \
    dnf install --installroot /mnt/rootfs --setopt=reposdir=/etc/yum.repos.d/ ${DNF_FLAGS} \
        glibc-minimal-langpack libgcc libstdc++ zlib zulu21-jdk-headless && \
    dnf --installroot /mnt/rootfs clean all

# Clean up the new root folder to reduce its size.
RUN rm -rf /mnt/rootfs/var/cache/* /mnt/rootfs/var/log/dnf* /mnt/rootfs/var/log/yum.*

# Copy BusyBox and further required files into the new root folder.
RUN cp -R /etc/yum.repos.d/ubi.repo /mnt/rootfs/etc/yum.repos.d/
COPY --from=busybox /bin/busybox /mnt/rootfs/bin/
RUN /mnt/rootfs/bin/busybox --install /mnt/rootfs/bin/

# Build the final container image
FROM scratch
COPY --from=builder /mnt/rootfs/ /
ENV JAVA_HOME=/opt/java PATH="/opt/java/bin/:${PATH}"
ENTRYPOINT ["/bin/sh", "/opt/run-java.sh"]
```

On that image, Trivy does identify the following installed components:

* basesystem
* bash
* filesystem
* glibc
* glibc-common
* glibc-minimal-langpack
* gpg-pubkey
* libgcc
* libstdc++
* ncurses-base
* ncurses-libs
* openjdk  <--- check
* redhat-release
* setup
* tzdata
* zlib

These are exactly the installed packages except for BusyBox. Especially the Java package is now identified by Trivy.
The image is equally small with a size of around ??? MB and has only minor violations (as of writing).

For discovering a more complete set of packages including BusyBox, consider [Grype](https://github.com/anchore/grype) as alternative to Trivy.
The related tool [Syft](https://github.com/anchore/syft) even allows to generate an SBOM (Software Bill of Materials) for the container image, a compliance requirement with growing attention.

The custom Java container image is now a good starting point for cloud-native Java applications.
It is small, secure, and nearly production-ready, if standard hardening approaches such as a non-root user are enforced.
The image can be further customized by adding additional tools, libraries, or configuration files as required.
For instance, the image can be extended with agents for observability, such as [APM](https://www.elastic.co/guide/en/apm/agent/java/current/index.html) or [Pyroscope](https://grafana.com/oss/pyroscope/).
Many more possibilities exist to tailor the image to the specific requirements of Java applications.

For guidelines and best practices concerning Dockerfiles, see https://docs.docker.com/develop/develop-images/dockerfile_best-practices/
