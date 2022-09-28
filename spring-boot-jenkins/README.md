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
4. Link podman socket to docker socket, and Mount the docker socket to new deployed container containing the docker executable
```shell
systemctl --user enable podman.socket  --now
sudo ln -s /run/user/${UID}/podman/podman.sock /var/run/docker.sock
podman run  --privileged -d --network host  -v /var/run/docker.sock:/var/run/docker.sock --name docker docker sleep infinity
```

5. Use maven jenkins image: maven:3.8.1-adoptopenjdk-11

6. In the pipeline, run the following stage(example):
```groovy
     stage("build & SonarQube analysis") {
          node {
              withSonarQubeEnv('My SonarQube Server') {
                 sh 'mvn clean package sonar:sonar'
              }
          }
      }

      stage("Quality Gate"){
          timeout(time: 1, unit: 'HOURS') {
              def qg = waitForQualityGate()
              if (qg.status != 'OK') {
                  error "Pipeline aborted due to quality gate failure: ${qg.status}"
              }
          }
      }
      
```
