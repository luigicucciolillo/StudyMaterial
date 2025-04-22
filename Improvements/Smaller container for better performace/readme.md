# Building small containers

- [Building small containers](#building-small-containers)
  - [Links](#links)
  - [Intro](#intro)
  - [Using Small Base Images](#using-small-base-images)
  - [Using the Builder Pattern](#using-the-builder-pattern)
  - [Performance and Secutiry Benefits](#performance-and-secutiry-benefits)
    - [Performance](#performance)
      - [Performance - Building the container](#performance---building-the-container)
      - [Performance - Pushing the container](#performance---pushing-the-container)
      - [Pull the container](#pull-the-container)
      - [Performance - Pulling the container](#performance---pulling-the-container)
    - [Security](#security)

## Links
**From Google Cloud Tech**:
- Getting Started on Google Cloud Platform: [link playlist](https://www.youtube.com/playlist?list=PLIivdWyY5sqL8biXl2L_axB8TpROQMkbQ)
- Building small containers: [link video](https://www.youtube.com/watch?v=wGz_cbtCiEA&list=PLIivdWyY5sqL8biXl2L_axB8TpROQMkbQ)
- Kubernetes best practices: [link playlist](https://www.youtube.com/watch?v=wGz_cbtCiEA&list=PLIivdWyY5sqL3xfXz5xJvwzFW_tlQB_GB)

## Intro
Let's explore how you can create small and secure container images. Thanks to Docker, creating container images has never been simpler. 
Specify your base image:
```yml
From node:onbuild
```
Add your changes:
```yml
EXPOSE 8080
```
And build your container:
```bash
docker build myapp .
```

While this is great for getting started, using the default base images can lead to large images full of security vulnerabilities. Most Docker images use Debian or Ubuntu as the base image. While this is great for compatibility and easy onboarding, **_these base images can add hundreds of megabytes of additional overhead to your container_**.

For example, simple Node.js and Go "Hello World" apps are around 700 megabytes (see image below). Your application is probably only a few megabytes in size. So all this additional overhead is wasted space and a great hiding place for security vulnerabilities and bugs.


<p align="center">
  <img src=image.png alt="Centered Image" width="600">
</p

[go and node size](image.png)

So let's look at two methods to reduce the container image size:
- Using small base images
- Using the builder pattern

## Using Small Base Images

Using smaller base images is probably the easiest way to reduce your container size. **_Chances are your language or stack provides an official image that's much smaller than the default image_**.

For example, let's take a look at our Node.js container (see image below). Going from the default `node:8` to `node:8-alpine` reduces our base image size by 10 times.

<p align="center">
  <img src=image-1.png alt="Centered Image" width="400">
</p>

[alt text](image-1.png)

To move to a smaller base image, update your Dockerfile to start with a new base image. Unlike the old `onbuild` image, you’ll need to copy your code into the container and install any dependencies:

```dockerfile
FROM node:alpine
WORKDIR /app
COPY package.json /app/package.json
RUN npm install --production
COPY server.js /app/server.js
EXPOSE 8080
CMD npm start
```

- In the new Dockerfile, the container starts with the alpine image
```dockerfile
FROM node:alpine
``` 
- creates a directory for the code 
```dockerfile
WORKDIR /app
```
- installs dependencies with npm 
```dockerfile
COPY package.json /app/package.json
RUN npm install --production
```
- and finally starts the Node.js server:
```dockerfile
COPY server.js /app/server.js
EXPOSE 8080
CMD npm start
```
With this update, the resulting container is almost 10 times smaller.

> _If your programming language or stack doesn't have an option for a small base image_, **you can build your container using raw Alpine Linux as a starting point**.  also gives you complete control over what goes inside your containers.

## Using the Builder Pattern

Now, using a small base image is a great way to quickly build small containers. But you might be able to _**go even smaller using the builder pattern**_.

With interpreted languages, the source code is sent to an interpreter and then executed directly. But with a compiled language, the source code is turned into compiled code beforehand. 

<p align="center">
  <img src=image-2.png alt="Centered Image" width="500">
</p>

[alt text](image-2.png)

The compilation step often requires tools that are not needed to actually run the code. So this means that you can remove these tools from the final container completely. To do this, use the builder pattern.

<p align="center">
  <img src=image-3.png alt="Centered Image" width="700">
</p>

[alt text](image-3.png)

The code is built in the first container, and then the compiled code is packaged in the final container **without all the compilers and tools required to make the compiled code**.

Let’s take a Go application through this process. First, move from the `onbuild` image:

```dockerfile
FROM go:onbuild
EXPOSE 8080
```

to Alpine Linux:

```dockerfile
FROM golang:alpine
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp
EXPOSE 8080 
ENTRYPOINT ./goapp
```

In the new Dockerfile, the container starts with a `golang:alpine` image. Then it creates a directory for the code, copies in the source code, builds the source, and finally starts the app. This container is much smaller than the `onbuild` container, _but it still contains a compiler and other Go tools that we don't really need_. 

So let’s extract just the compiled program and put it into its own container:

```dockerfile
#Section 1 **********************
FROM golang:alpine AS build-env
WORKDIR /app
ADD . /app
RUN cd /app && go build -o goapp
#Section 2 **********************
FROM alpine
RUN apk update && apk add ca- certificate && rm -rf /var/cache/apk/*
WORKDIR /app
COPY --from=build-env /app/goapp /app

EXPOSE 8080 
ENTRYPOINT ./goapp
```



You might notice something strange about this Dockerfile—it has two `FROM` lines. 
- The first section looks exactly the same as before, except that _it uses the `AS` keyword to give this step a name_.
- In the next section (Section 2), there is a new `FROM` line. This starts a fresh image, and instead of using `golang:alpine`, we use raw Alpine as the base image. 

Raw Alpine Linux doesn’t have any SSL certificates installed, _**which will make most API calls over HTTPS fail**_. So **let’s install** some root CA certificates. (RUN ... in section 2)

Now comes the interesting part: 
- you can use the `COPY` command to copy the compiled code from the first container into the second.

```dockerfile
COPY --from=build-env /app/goapp /app
```

This line _copies just that one file_, and _not the rest of the Go tooling_.

This new multi-stage Dockerfile creates a container image that’s just **_12 megabytes_**. The original container image was **_700 megabytes_**.  That is quite a difference.

> Using _small base images_ and the _builder pattern_ are great ways to create much smaller containers without a lot of work.

## Performance and Secutiry Benefits

Now, depending on your application stack, there may be additional ways to reduce your container image size. But do small containers actually have a measurable advantage?

Let’s look at two areas where small containers shine: **performance** and **security**.

### Performance

First, let’s look at how long it takes to:
- Build a container
- Push it to a registry
- Pull it down from the registry

#### Performance - Building the container
type  | Large machine | small machine
--- | --- | ---
go:onbuild | 35 seconds | 54 seconds
go:multistage (builder_pattern)| 23 seconds | 28 seconds

For the initial build, smaller containers have a huge advantage over larger containers. Docker caches layers, so subsequent builds are faster. But many CI systems don’t cache layers, so time savings add up.

#### Performance - Pushing the container
type  | Large machine | small machine
--- | --- | ---
go:onbuild | 15 seconds | 48 seconds
go:multistage (builder_pattern)| 14 seconds | 16 seconds

Now that the container is built, you need to push it to a container registry. 

> Advisement from Google:
I recommend using the Google Container Registry (GCR). You only pay for the raw storage and network—no additional management fees. It’s private, secure, and lightning fast. GCR uses many tricks to speed up pushing. For example, push times for large and small containers can be similar because GCR uses a global cache for common base images, meaning you don’t need to upload them at all. With small machines, the CPU becomes the bottleneck, but even then, there is still a significant advantage to using small containers. If you’re using GCR, I also recommend using Google Container Builder as part of your build system. It’s faster than even a large machine, and you get 120 free build minutes per day, which should cover most container-building needs.

#### Pull the container

Now, the most important performance metric: **pulling the container**. Even if build and push times don’t matter to you, pull times definitely should.
Let’s say one node in your three-node Kubernetes cluster crashes.

<p align="center">
  <img src=image-4.png alt="Centered Image" width="400">
</p>

[alt text](image-4.png)

If you're using Google Kubernetes Engine, a new node will automatically spin up to take its place. But this new node is completely fresh and will have to pull all your containers before it can start working.

<p align="center">
  <img src=image-5.png alt="Centered Image" width="400">
</p>

[alt text](image-5.png)

If it takes too long to pull containers, your cluster underperforms. This can also happen when:
- Adding new nodes
- Upgrading nodes
- Deploying new containers

Minimizing pull times is key. Smaller containers pull much faster than large ones, and with multiple containers per node, this time adds up quickly. **_Using small, common base images, significantly speeds up deployment and the speed at which new Kubernetes nodes come online._**

#### Performance - Pulling the container
type  | Large machine | small machine
--- | --- | ---
go:onbuild | 26 seconds | 52 seconds
go:multistage (builder_pattern)| 6 seconds | 6 seconds

### Security

As golden rule we often say that smaller containers are more secure because they have less surface area for attacks.
Google Container Registry can automatically scan your containers for vulnerabilities. I scanned both the `onbuild` and multi-stage containers I built a few months ago.

<p align="center">
  <img src=image-6.png alt="Centered Image" width="700">
</p>

[alt text](image-6.png)

The results?  
- **Small container:** 3 medium vulnerabilities  
- **Large container:** 16 critical and over 370 total vulnerabilities  

And most issues in the large container had nothing to do with the app, but were from programs we weren’t even using.

<p align="center">
  <img src=image-7.png alt="Centered Image" width="900">
</p>

[alt text](image-7.png)

**_When people talk about increased attack surface, this is what they mean._**

---

> **So remember: Build small containers. The performance and security benefits are real.**  