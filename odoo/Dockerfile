FROM lasley/docker-odoo-base
MAINTAINER LasLabs Inc <support@laslabs.com>

VOLUME /home/odoo/.ssh

# Metadata
ARG BUILD_DATE
ARG VCS_REF
ARG VERSION
LABEL org.label-schema.build-date=$BUILD_DATE \
      org.label-schema.name="LasLabs.com Runbot v9" \
      org.label-schema.description="Provides an Odoo environment for the LasLabs Production Website." \
      org.label-schema.url="https://laslabs.com/" \
      org.label-schema.vcs-ref=$VCS_REF \
      org.label-schema.vcs-url="https://repo.laslabs.com/scm/dep/laslabs.com.git" \
      org.label-schema.vendor="LasLabs Inc." \
      org.label-schema.version=$VERSION \
      org.label-schema.schema-version="1.0"
