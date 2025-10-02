# Jenkins Agent with Python

Custom Jenkins agent Docker image based on `jenkins/agent:latest-jdk17` with Python 3 installed.

## Building the Image

```bash
docker build -t <your-username>/jenkins-agent-python:latest .
```

## Pushing to Docker Hub

1. Login to Docker Hub:
```bash
docker login
```

2. Push the image:
```bash
docker push <your-username>/jenkins-agent-python:latest
```

## Usage

Use this image as a Jenkins agent in your pipeline or node configuration to run jobs that require Python.
