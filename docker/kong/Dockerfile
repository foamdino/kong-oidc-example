FROM kong:2.1-centos

LABEL description="Centos 7 + Kong 2.1 + kong-oidc plugin"

USER root

RUN yum install -y git unzip && yum clean all

RUN luarocks install kong-oidc

USER kong
