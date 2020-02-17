To perform an easy deployment of the Odoo environment with persistent Postgresql database you can use the template (template.yaml)

The image was built with the aim of raising a productive Odoo environment in the Openshift Container Platform.

To achieve this we cloned the official odoo repository for docker environments:
```
https://github.com/odoo/docker
```

Then we modified the Dockerfile file of version 12 with the aim of complying with the good practices and recommendations necessary so that these images could run in Openshift.

As the images in Openshift run with an arbitrary user that belongs to the root group, it was necessary to create an additional script to create that user in the file / etc / passwd.
**uid_entrypoint**
```
#!/bin/sh
if ! whoami &> /dev/null; then
  if [ -w /etc/passwd ]; then
    echo "${USER_NAME:-default}:x:$(id -u):0:${USER_NAME:-default} user:${HOME}:/sbin/nologin" >> /etc/passwd
  fi
fi
exec "$@"
```

Therefore, the Dockerfile file remains as follows:
```
FROM debian:stretch
LABEL maintainer="Odoo S.A. <info@odoo.com>"

# Generate locale C.UTF-8 for postgres and general locale data
ENV LANG C.UTF-8

# Install some deps, lessc and less-plugin-clean-css, and wkhtmltopdf
RUN set -x; \
        apt-get update \
        && apt-get install -y --no-install-recommends \
            ca-certificates \
            curl \
            dirmngr \
            fonts-noto-cjk \
            gnupg \
            libssl1.0-dev \
            node-less \
            python3-pip \
            python3-pyldap \
            python3-qrcode \
            python3-renderpm \
            python3-setuptools \
            python3-vobject \
            python3-watchdog \
            xz-utils \
        && curl -o wkhtmltox.deb -sSL https://github.com/wkhtmltopdf/wkhtmltopdf/releases/download/0.12.5/wkhtmltox_0.12.5-1.stretch_amd64.deb \
        && echo '7e35a63f9db14f93ec7feeb0fce76b30c08f2057 wkhtmltox.deb' | sha1sum -c - \
        && dpkg --force-depends -i wkhtmltox.deb\
        && apt-get -y install -f --no-install-recommends \
        && rm -rf /var/lib/apt/lists/* wkhtmltox.deb

# install latest postgresql-client
RUN set -x; \
        echo 'deb http://apt.postgresql.org/pub/repos/apt/ stretch-pgdg main' > etc/apt/sources.list.d/pgdg.list \
        && export GNUPGHOME="$(mktemp -d)" \
        && repokey='B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8' \
        && gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "${repokey}" \
        && gpg --armor --export "${repokey}" | apt-key add - \
        && gpgconf --kill all \
        && rm -rf "$GNUPGHOME" \
        && apt-get update  \
        && apt-get install -y postgresql-client \
        && rm -rf /var/lib/apt/lists/*

# Install rtlcss (on Debian stretch)
RUN set -x;\
    echo "deb http://deb.nodesource.com/node_8.x stretch main" > /etc/apt/sources.list.d/nodesource.list \
    && export GNUPGHOME="$(mktemp -d)" \
    && repokey='9FD3B784BC1C6FC31A8A0A1C1655A0AB68576280' \
    && gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "${repokey}" \
    && gpg --armor --export "${repokey}" | apt-key add - \
    && gpgconf --kill all \
    && rm -rf "$GNUPGHOME" \
    && apt-get update \
    && apt-get install -y nodejs \
    && npm install -g rtlcss \
    && rm -rf /var/lib/apt/lists/*

# Install Odoo
ENV ODOO_VERSION 12.0
ARG ODOO_RELEASE=20190128
ARG ODOO_SHA=9e34aaed2eb1e7697aaf36767247dbf335e9fe7a
RUN set -x; \
        curl -o odoo.deb -sSL http://nightly.odoo.com/${ODOO_VERSION}/nightly/deb/odoo_${ODOO_VERSION}.${ODOO_RELEASE}_all.deb \
        && echo "${ODOO_SHA} odoo.deb" | sha1sum -c - \
        && dpkg --force-depends -i odoo.deb \
        && apt-get update \
        && apt-get -y install -f --no-install-recommends \
        && rm -rf /var/lib/apt/lists/* odoo.deb

# Copy entrypoint script and Odoo configuration file
RUN pip3 install num2words xlwt
COPY ./entrypoint.sh /
COPY ./uid_entrypoint /
COPY ./odoo.conf /etc/odoo/
RUN chown odoo:0 /etc/odoo/odoo.conf \
        && chgrp -R 0 /var/lib/odoo \
        && chmod -R g=u /var/lib/odoo \
        && chmod g=u /etc/passwd

# Mount /var/lib/odoo to allow restoring filestore and /mnt/extra-addons for users addons
RUN mkdir -p /mnt/extra-addons \
        && chown -R odoo:0 /mnt/extra-addons \
        && chmod -R g=u /mnt/extra-addons
VOLUME ["/var/lib/odoo", "/mnt/extra-addons"]

# Expose Odoo services
EXPOSE 8069 8071

# Set the default config file
ENV ODOO_RC /etc/odoo/odoo.conf

# Set default user when running the container
USER 100001

ENTRYPOINT ["/uid_entrypoint"]

CMD ["/entrypoint.sh","odoo"]
```

For the environment of productive odoo with its postgres databases to function correctly we must:

Firstly, carry out a deployment of a Postgres database version >= 9.4 with persistent volumes and the parameter **POSTGRES_DB=postgres**
Then connect them to the pods where the database is running and give permission to the user defined by environment variable or by default   **odoo**  so you can create databases as the following example demonstrates or through the Openshift visual interface.
```
# oc rsh db-1-s8zw1
# psql
$ ALTER ROLE [odoo] WITH CREATEDB;
$ \q
# exit
```
**Note:** The user must be the one configured in the environment variable when instantiating the odoo image for Openshift
