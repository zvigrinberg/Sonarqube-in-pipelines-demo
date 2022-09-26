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

4. Use maven jenkins slave on [Container image](https://hub.docker.com/r/bibinwilson/jenkins-slave/)

5. In the pipeline, run the following stage(example):
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
