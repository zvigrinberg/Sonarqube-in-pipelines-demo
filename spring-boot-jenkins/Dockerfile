FROM redhat/ubi8:latest
USER root
RUN yum install -y maven
RUN useradd appuser
USER appuser
RUN mkdir -p /home/appuser
CMD cat
