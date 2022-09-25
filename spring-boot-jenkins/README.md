# Run Sonarqube Scanner with Jenkins Pipeline

## Pre-requisites:

1. Create network in podman/docker
```shell
podman network create jenkins-sonarqube
```
2. Run SonarQube Instance on docker/podman:
```shell
podman run -d --network=jenkins-sonarqube --name sonarqube -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true -p 9000:9000 sonarqube:latest
```

3. Run jenkins instance on docker/podman:
```shell
podman run -d -p 8080:8080 -p 50000:50000 --network=jenkins-sonarqube --restart=on-failure jenkins/jenkins:latest
```