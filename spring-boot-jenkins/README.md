# Run Sonarqube Scanner with Jenkins Pipeline Using Docker Pipeline Plugin.

## Objectives
  - To Provide a common generic way to run pipeline shell commands in any container
  - To run Sonarqube scanner in jenkins using a maven container, and a sonarqube plugin that integrates with a sonarqube server.

## Procedure:

1. Create network in podman/docker
```shell
podman network create jenkins-sonarqube
```
2. Run SonarQube Instance on docker/podman:
```shell
podman run -d --network=jenkins-sonarqube --name sonarqube -e SONAR_ES_BOOTSTRAP_CHECKS_DISABLE=true -p 9000:9000 sonarqube:latest
```

3. We'll use docker-pipeline plugin in Jenkins, in order to run agent with command inside whatever container we want, in order to achieve that , we need to make sure that filesystem type of jenkins container is the same as the file system of docker server container, otherwise jenkins won't be able to mount workspace files into container, because file systems are incompatible , so the only 2 ways to guarantee it, is to run docker daemon in same jenkins master/agent, or to create a shared volume on the host and mount it both to jenkins container and docker server container, We'll use the first approach:
   
   1. Create new Volume on host 
   ```shell
    podman volume create jenkins-data
   ```
   2. If you are not root, open new user namespace as root
   ```shell
    podman unshare
   ```
   3. get the path of the shared volume on host by running the following command: 
   ```shell
    export shareDirectory=$(podman volume inspect jenkins-data | jq .[].Mountpoint)
   ```
   4. Give full permissions to directory for containers to be able to read,write the shared volume:
   ```shell
   chmod -R 777 $shareDirectory
   ```
   5. Exit from new user namespace
   ```shell
   exit
   ```
 
  
4. Run Docker server instance on docker/podman:
```shell
podman run   --name docker-server  --detach   --privileged   --network jenkins-sonarqube --hostname=my-docker  --network-alias docker   --env DOCKER_TLS_CERTDIR=/certs   --volume jenkins-docker-certs:/certs/client   --volume jenkins-data:/var/jenkins_home   --publish 2376:2376   docker:dind
```

[comment]: <> (6. Link podman socket to docker socket, and Mount the docker socket to new deployed container containing the docker executable)

[comment]: <> (```shell)

[comment]: <> (systemctl --user enable podman.socket  --now)

[comment]: <> (sudo ln -s /run/user/${UID}/podman/podman.sock /var/run/docker.sock)

[comment]: <> (podman run  --privileged -d --network host  -v /var/run/docker.sock:/var/run/docker.sock --name docker docker sleep infinity)

[comment]: <> (```)
5. Transplant A DNS record in jenkins container that will be mapped to IP's of docker server and sonarqube server in network jenkins-sonarqube 
   ```shell
   ## get ips address of docker server and sonarqube server in network jenkins-sonarqube 
    export DOCKER_SERVER=$(podman inspect docker-server | jq '.[].NetworkSettings.Networks."jenkins-sonarqube".IPAddress' | awk -F \" '{print $2}')
    export SONARQUBE_SERVER=$(podman inspect sonarqube | jq '.[].NetworkSettings.Networks."jenkins-sonarqube".IPAddress' | awk -F \" '{print $2}')
   ## transplant a dns record my-docker  for the ip of docker server in jenkins instance container - better to do with --add-host when will run the jenkins container
    podman exec --privileged --user=root jenkins-server sed -i -c  '$ a '$DOCKER_SERVER'      my-docker' /etc/hosts
   ```
6. Run jenkins instance on docker/podman:
  ```shell
  podman run -d -p 8080:8080 -p 50000:50000 --network=jenkins-sonarqube --env DOCKER_HOST=tcp://docker:2376 --env DOCKER_CERT_PATH=/certs/client --env   DOCKER_TLS_VERIFY=1 --volume jenkins-data:/var/jenkins_home --volume jenkins-docker-certs:/certs/client:ro --add-host my-docker:$DOCKER_SERVER --add-host sonarqube:$SONARQUBE_SERVER --name jenkins-server --restart=on-failure  jenkins/jenkins:latest
  ```
   - For jenkins required Configuration, [follow this](./Jenkins-README.md)    
7. Use image containing maven - built from section 4

8. In the pipeline, run the following stage(example):
```groovy
node {
    stage("build & SonarQube analysis") {
        withSonarQubeEnv('sonar-qube') {
            docker.withTool('docker-test'){
                docker.withServer('tcp://my-docker:2376','docker-server-certs'){

                    docker.image('quay.io/zgrinber/maven:test3').inside{

                        sh 'mvn clean package sonar:sonar'
                    }
                }
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

}

      
```
