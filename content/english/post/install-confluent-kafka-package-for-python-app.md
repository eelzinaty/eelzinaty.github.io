+++
author = "Issam Alzinati"
title = "How to dockerize and optimize python applications that depend on librdkafka library running on Mac M1"
date = "2022-07-28"
description = "How to dockerize and optimize python application that depend on librdkafka library running on Mac M1"
tags = [
    "python",
    "docker",
    "kafka",
]
+++

Recently I was working on a Python(3.7) project that uses Kafka. I added the Python package, `confluent-kafka` because we are using hosted Confluent Kafka, to the requirements file and created a simple docker file that installs the requirements, nothing fancy. This setup runs without any issues in our CI/CD, GitHub Actions, and images were built and pushed seamlessly.

```Dockerfile
FROM python:3.7-buster
WORKDIR /app
COPY . /app
RUN pip install -r requirements.txt
RUN chmod +x /app/start-service.sh
EXPOSE 8000
CMD ["/app/start-service.sh"]
```

When I tried to build the docker image locally in my machine, Mac M1 Monterey, I ran into errors related to `librdkafka` not being found when installing `confluent-kafka` package!

```shell
#15 68.10       In file included from /tmp/pip-install-dd5d18da/confluent-kafka_0ff22487b1fc4160adbd556f0bfa187f/src/confluent_kafka/src/confluent_kafka.c:17:
#15 68.10       /tmp/pip-install-dd5d18da/confluent-kafka_0ff22487b1fc4160adbd556f0bfa187f/src/confluent_kafka/src/confluent_kafka.h:23:10: fatal error: librdkafka/rdkafka.h: No such file or directory
#15 68.10        #include <librdkafka/rdkafka.h>
#15 68.10                 ^~~~~~~~~~~~~~~~~~~~~~
#15 68.10       compilation terminated.
#15 68.10       error: command 'gcc' failed with exit status 1
#15 68.10       [end of output]
#15 68.10   
#15 68.10   note: This error originates from a subprocess, and is likely not a problem with pip.
#15 68.10 error: legacy-install-failure
#15 68.10 
#15 68.10 × Encountered error while trying to install package.
#15 68.10 ╰─> confluent-kafka
```

The `confluent-kafka` package is [high level Kafka client for Python](https://github.com/confluentinc/confluent-kafka-python), and it is based on `librdkafka`. `librdkafka` is [the Apache Kafka C/C++ client library](https://github.com/edenhill/librdkafka) that provides the messages producing and consuming functionalities for applications that use Apache Kafka for async communications and event streaming. 

Interestingly, why a package that has external dependency would fail?! It is working on the CI/CD, `Linux Ubuntu`, why not on my machine(Mac M1)?! I tried to build the image in different Mac M1 machines and got the same error. But isn't docker suppose to build and run anywhere despite the underlying host?!

After hours of googling, trying different settings, and approaches... I came across this [excellent post](https://pythonspeed.com/articles/docker-build-problems-mac/) that explains in detail why specific packages are failing on Mac M1&M2 machines. 

**Issue TL;DR**
> Mac M1&M2 machines are ARM based. When the docker image is pulled, docker checks the underlying CPU architecture, then either pulls AMD or ARM images. Some python packages are only supporting AMD architecture, x86_64. To fix this issue you need either to pull the source code and build it or pull the docker image with ARM architecture.


## Solution
I decided to go with installing and building the source code. Checking `librdkafka` [build from source section](https://github.com/edenhill/librdkafka#build-from-source), I created the following docker file:

```Dockerfile
FROM python:3.7-buster as base

# Downlaod the librdkafka source code
RUN git clone  --depth 1 --branch v1.9.1 https://github.com/edenhill/librdkafka.git
WORKDIR librdkafka

# Installation steps based on librdkafka build from source instructions
RUN ./configure --prefix /usr
RUN make
RUN make install

# Try to install confluent-kafka package
RUN pip3 install --upgrade pip
RUN python3 -m pip install --no-binary confluent-kafka confluent-kafka

# Verify that confluent_kafka is working without issues
RUN python3 -c 'import confluent_kafka; print(confluent_kafka.version())'
```

That is it! The image was built successfully.


## Bonus Tip: Optimizing the solution by using better docker caching and reducing the image size

There is only one problem here: adding downloading and building this image will add extra steps to the build process, and it will increase the size of the image.

To solve this, we can use a multistage docker file structure and enable docker [buildkit](https://github.com/moby/buildkit) for parallel executing and enhanced caching.

### Containerizing the application without optimization
The original Dockerfile looked like this:
```Dockerfile
FROM python:3.7-buster as base

# Downlaod the librdkafka source code
RUN git clone  --depth 1 --branch v1.9.1 https://github.com/edenhill/librdkafka.git
WORKDIR librdkafka

# Installation steps based on librdkafka build from source instructions
RUN ./configure --prefix /usr
RUN make
RUN make install

WORKDIR /app

# Installing python packages requirements.txt
COPY requirements.txt requirements.txt
RUN pip3 install --upgrade pip
RUN pip3 install wheel && pip3 install -r requirements.txt

# Get the code base and start the application
COPY . /app
RUN chmod +x /app/start-server.sh
CMD ["sh", "/app/start-server.sh"]
```

The first build took about 5 minutes and the image size was `1.57GB`.
```shell
$ docker image ls                    
REPOSITORY          TAG          IMAGE ID       CREATED              SIZE
platform-testing    test         8fe8cb338ca7   About a minute ago   1.57GB
```

### Containerizing the application optimization(Faster Build and Smaller Image)

We are going to do the following:
1. Enable docker [buildkit](https://github.com/moby/buildkit) for parallel executing and enhanced caching. This will speed up the build time.
```shell
export DOCKER_BUILDKIT=1
```
2. Change the docker file into a multistage build. This will decrease the image size.
3. Using a lighter image at the final stage to run the application. For Python, you can use images with `-slim` build.

```Dockerfile
# The first stage is for installing the librdkafka library
FROM python:3.7-buster as base

## virtualenv setup
ENV VIRTUAL_ENV=/opt/venv
RUN python3 -m venv $VIRTUAL_ENV
ENV PATH="$VIRTUAL_ENV/bin:$PATH"

# Downlaod the librdkafka source code
RUN git clone  --depth 1 --branch v1.9.1 https://github.com/edenhill/librdkafka.git
WORKDIR librdkafka

# Installation steps based on librdkafka build from source instructions
RUN ./configure --prefix /usr
RUN make
RUN make install


# The second stage is for installing application dependancy
FROM base as builder

WORKDIR /app

# Installing python packages requirements.txt
COPY requirements.txt requirements.txt
RUN pip3 install --upgrade pip
RUN pip3 install wheel && pip3 install -r requirements.txt


# The third stage is using slim image build and only runs the application
FROM python:3.7-slim-buster as runtime

# Copy the installed dependancies and setup the virtual environment
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Get the code base and start the application
COPY . /app
RUN chmod +x /app/start-server.sh
CMD ["sh", "/app/start-server.sh"]
```

The first build took about 3.5 minutes and the image size was `354MB`, about `75%` decrease in size.
```shell
$ docker build -f Dockerfile.optimized . -t platform-testing:optimized
[+] Building 200s (22/22) FINISHED

$ docker image ls                                                                                 
REPOSITORY          TAG               IMAGE ID       CREATED          SIZE
platform-testing    optimized         60647c7682e0   44 seconds ago   354MB
```
