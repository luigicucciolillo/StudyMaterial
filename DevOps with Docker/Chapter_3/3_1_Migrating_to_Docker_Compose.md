---
title: 'Migrating to Docker Compose'
---

# 3.1 Migrating to Docker Compose

Even with a simple image, we've already been dealing with plenty of command line options in both building, pushing and running the image.

Next we will switch to a tool called [Docker Compose](https://docs.docker.com/compose/) to manage these. Docker Compose used to be a separate tool but now it is integrated into Docker and can be used like the rest of the Docker commands.

Docker Compose is designed to simplify running multi-container applications using a single command.

Assume that we are in the folder where we have our Dockerfile with the following content:

```dockerfile
FROM ubuntu:24.04

WORKDIR /mydir

RUN apt-get update && apt-get install -y curl python3
RUN curl -L https://github.com/yt-dlp/yt-dlp/releases/latest/download/yt-dlp -o /usr/local/bin/yt-dlp
RUN chmod a+x /usr/local/bin/yt-dlp

ENTRYPOINT ["/usr/local/bin/yt-dlp"]
```

Let us now create a file called `docker-compose.yml`:

```yaml
services:
  yt-dlp-ubuntu:
    image: <username>/<repositoryname>
    build: .
```

File defines now one service that is given name yt-dlp-ubuntu.

The value of the key `build` can be a file system path (in the example it is the current directory `.`) or an object with keys `context` and `dockerfile`, see the [documentation](https://docs.docker.com/compose/compose-file/build/) for more

Now we can build and push with just these commands:

```sh
$ docker compose build
$ docker compose push
```

## Volumes in Docker Compose

To run the image as we did previously, we will need to add the volume bind mounts. [Volumes in Docker Compose](https://docs.docker.com/engine/storage/volumes/#use-a-volume-with-docker-compose) are defined with the following syntax `location-in-host:location-in-container`. Compose can work without an absolute path:

```yaml
services:
  yt-dlp-ubuntu:
    image: <username>/<repositoryname>
    build: .
    volumes:
      - .:/mydir
    container_name: yt-dlp
```

We can also give the container a name it will use when running with container_name. The service name can be used to run it:

```sh
$ docker compose run yt-dlp-ubuntu https://imgur.com/JY5tHqr
```

## Using a ready image with Docker Compose

In the previous example, we used the key `build` to tell how the `image` defined in the compose file is built if the command `docker compose build` is used. It is pretty common that we use some readily `built` images, and in that case, the key build is not needed. We could eg, use the following docker-compose.yaml file to define two containers, one based on image [nginx:1.27](https://hub.docker.com/_/nginx) and the other based on [postgres:17](https://hub.docker.com/_/postgres):

```
services:
  nginx:
    image: nginx:1.27
  database:
    image: postgres:17
```

Both of the containers can now be started with command `docker compose up`:

```sh
nginx-1     | 2025/02/28 13:05:23 [notice] 1#1: start worker process 28
nginx-1     | 2025/02/28 13:05:23 [notice] 1#1: start worker process 29
nginx-1     | 2025/02/28 13:05:23 [notice] 1#1: start worker process 30
nginx-1     | 2025/02/28 13:05:23 [notice] 1#1: start worker process 31
database-1  | Error: Database is uninitialized and superuser password is not specified.
database-1  |        You must specify POSTGRES_PASSWORD to a non-empty value for the
database-1  |        superuser. For example, "-e POSTGRES_PASSWORD=password" on "docker run".
database-1  |
database-1  |        You may also use "POSTGRES_HOST_AUTH_METHOD=trust" to allow all
database-1  |        connections without a password. This is *not* recommended.
database-1  |
database-1  |        See PostgreSQL documentation about "trust":
database-1  |        https://www.postgresql.org/docs/current/auth-trust.html
database-1 exited with code 1
```

The nginx web server starts up but as we can see from the log output the Postgres database does not start due to some missing configurations. However, we shall not care about that for now.

## Key Commands in Docker Compose

Before going to exercises, let us go through the commands that you need most often:

To start all the services defined in the `docker-compose.yaml` file:

```sh
$ docker compose up
```

To stop and remove the running services:

```sh
$ docker compose down
```

If you want to monitor the output of the running containers and debug issues, we can view the logs with:

```sh
$ docker compose logs
```

To lists all the services along with their current status:

```sh
$ docker compose ps
```

Full list of Compose CLI Commands can be found [here](https://docs.docker.com/reference/cli/docker/compose/).

---

## Exercise 2.1

Let us now leverage the Docker Compose with the simple webservice that we used in the [Exercise 1.3](/part-1/section-2#exercise-13)

Without a command `devopsdockeruh/simple-web-service` will create logs into its `/usr/src/app/text.log`.

Create a docker-compose.yml file that starts `devopsdockeruh/simple-web-service` and saves the logs into your filesystem.

Submit the docker-compose.yml, and make sure that it works simply by running `docker compose up` if the log file exists.

---

## Web services in Docker Compose

Compose is really meant for running web services, so let's move from simple binary wrappers to running a HTTP service.

<https://github.com/jwilder/whoami> is a simple service that prints the current container id (hostname).

```sh
$ docker container run -d -p 8000:8000 jwilder/whoami
  736ab83847bb12dddd8b09969433f3a02d64d5b0be48f7a5c59a594e3a6a3541
```

Navigate with a browser or curl to localhost:8000, they both will answer with the id.

Take down the container so that it's not blocking port 8000.

```sh
$ docker container stop 736ab83847bb
$ docker container rm 736ab83847bb
```

Let's create a new folder and a Docker Compose file `whoami/docker-compose.yml` from the command line options.

```yaml
services:
  whoami:
    image: jwilder/whoami
    ports:
      - 8000:8000
```

Test it:

```sh
$ docker compose up -d
$ curl localhost:8000
```

As you can see, the flag `-d` started the container in detached mode. When you are ready, you can stop and remove the container with `docker compose down`.

Environment variables can also be given to the containers in Docker Compose as follows:

```yaml
version: '3.8'

services:
  backend:
    image:
    environment:
      - VARIABLE=VALUE
      - VARIABLE2=VALUE2
```

Note that there are also [other](https://docs.docker.com/compose/environment-variables/set-environment-variables/), perhaps more elegant ways to define the environment variables in Docker compose.

---

## Exercise 2.2

Read about how to add the command to docker-compose.yml from the [documentation](https://docs.docker.com/compose/compose-file/compose-file-v3/#command).

The familiar image `devopsdockeruh/simple-web-service` can be used to start a web service, see the [exercise 1.10](/part-1/section-5#exercise-110).

Create a docker-compose.yml, and use it to start the service so that you can use it with your browser.

Submit the docker-compose.yml, and make sure that it works simply by running `docker compose up`

---

## Mandatory Exercise 2.3

As we saw previously, starting an application with two programs was not trivial and the commands got a bit long.

In the [previous part](/part-1/section-6) we created Dockerfiles for both [frontend](https://github.com/docker-hy/material-applications/tree/main/example-frontend) and [backend](https://github.com/docker-hy/material-applications/tree/main/example-backend) of the example application. Next, simplify the usage into one docker-compose.yml.

Configure the backend and front end to work in Docker Compose, as in [Exercise 1.14](https://courses.mooc.fi/org/uh-cs/courses/devops-with-docker/chapter-2/utilizing-tools-from-the-registry#9227044c-5b55-4b89-b568-fc5071166025). If the button for Exercise 1.14 works, your setup is correct.

You may assume that the images are readily built and available (locally or in Docker Hub).

Submit the docker-compose.yml

---