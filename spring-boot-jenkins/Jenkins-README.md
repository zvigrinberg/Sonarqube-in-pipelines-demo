# Jenkins Setup.

 - Define X.509 Client Certificate In Jenkins global credentials - Containing CA(Certificate Authority), Certificate , and private key, all resides in docker server(dind) container at path /certs/client, name it docker-server-certs.
 - Install Jenkins Sonarqube plugin And Docker Pipeline(If not already installed) - Restart Jenkins.
 - Go to Configure Jenkins --> Global Tool Configuration --> Install Docker --> name it docker-test --> add installer --> download from docker.com --> save.
 - Go to Configure Jenkins --> Configure system --> Search for sonarqube --> name it sonar-qube --> and populate all details of server ip/dns(with port 9000).
 
