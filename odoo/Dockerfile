FROM lasley/docker-odoo-base
MAINTAINER LasLabs Inc <support@laslabs.com>

VOLUME /home/odoo/.ssh

# Metadata
ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
LABEL org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.name="LasLabs.com Runbot v9" \
      org.label-schema.description="Provides a Dockerized Runbot." \
      org.label-schema.url="https://laslabs.com/" \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vcs-url="https://github.com/LasLabs/docker-runbot" \
      org.label-schema.vendor="LasLabs Inc." \
      org.label-schema.version=$VERSION \
      org.label-schema.schema-version="1.0"
