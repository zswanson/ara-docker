Please note, the docker image from this repo is for an older version of ARA; I unfortunately am consumed by work right now and don't have much time to update this for the newer 1.0 release which had significant architectural changes. 

# ara-docker
ARA (ARA Records Ansible) docker build, running on gunicorn

## Overview

I had a hard time figuring out how to install/configure the ARA server separately from my ansible execution host using apache and mod_wsgi. Turning to docker for a faster workflow, I tried out the apache-wsgi docker images which worked easily but created enormous images that seemed to run slow. GreenUnicorn (gunicorn) seemed to be a faster and lighterweight solution. This image provides an ARA server running in gunicorn with support for postgresql. Due to issues with mysql glibc dependencies under alpine, mysql isn't provided yet. Image size difference between the python alpine (musl) and python-slim (glibc) base images was around 200MB, so I'm sticking with this limited functionality until I see a workaround.

## External Dependencies
- Ansible client. The ARA server only displays the records of ansible playbook runs. Ansible must be executed elsewhere. This ansible installation must also include [ARA callback configuration](https://ara.readthedocs.io/en/latest/configuration.html#ansible) in order to post the playback reports to the database.
- External postgresql database. Required so that the ansible ARA callbacks can populate your playbook data to the database, and for ARA to read the data.

## Ports
The container exposes port `8000` and requires that to be mapped for client connections.

## External postgresql  (preferred method)
The image is pre-configured with python drivers for postgresql (psycopg2). Check the [ARA database documentation](https://ara.readthedocs.io/en/latest/configuration.html#ara-database) for any changes to the database setup. Install your database either as a real install or a linked container (docker-compose, etc). Provide the connection url as the `ARA_DATABASE` environment variable. The url must include relevant username and password if required. If you're running postgres in a container, be sure to expose its port, so that both ARA and the remote Ansible client can connect.

## Default sqlite database (not recommended)
If not overridden (see the [ARA documentation](https://ara.readthedocs.io/en/latest/configuration.html#ara-database)) the ARA container will default to the internal 'development' sqlite database. The sqlite schema will auto populate and requires no additional configuration. To persist the sqlite, mount an external volume to /ara. This is only suitable if you also reconfigure the container's installation of ansible and run ansible directly from the container. Otherwise there is is no way for an external ansible-playbook callback to reach the sqlite database, as sqlite is not a server! 


Provide the configured the postgresql environment variable as:
```sh
ARA_DATABASE=postgresql+psycopg2://username:password@dbhost:port/ara
```

where `dbhost` is either the real hostname/ip, or a container name if using container networking. If you don't provide a port the driver will use the relevant defaults.

## ARA server configuration

Not all the parameters for ARA apply to the web server, but any that do can be overriden either through a config file or environment variables passed to the container. See the [ARA documentation](https://ara.readthedocs.io/en/latest/configuration.html#parameters-and-their-defaults) for details. To override the config used for the ARA server, mount a volume to the container and override the `ANSIBLE_CONFIG` environment variable to point to your configuration file. Or mount over the /ara/ara.cfg file provided in the image. (Things may or may not work correctly if you don't set, or change, the ARA_DIR variable)

##
The container will write ARA logs to /ara/ara.log. If you map the /ara volume to a bind mount, you can persist the log and configurations through container lifecycles. gunicorn logs to stdout/stderr and can be captured through the docker interface.

## Sample docker command
```sh
docker run -p 8000:8000 -e ARA_DATABASE=postgresql+psycopg2://ara:ara@db:5432/ara ara:latest
```

## Sample docker-compose.yml

You'll need to pre-configure the db in postgresql as per the ARA documentation. One way to do that, for postgresql, is to mount a local init.sql that's used by the postgresql container entrypoint.

```yaml
version: '3'
services:
  ara:
    image: ara:latest
    ports:
      - 8000:8000
    environment:
      - ARA_DATABASE=postgresql+psycopg2://ara:ara@db:5432/ara
  db:
    image: postgres:11
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
```

You can also provide a postgres user, password and dbname as environment variables.

```yaml
version: '3'
services:
  db:
    image: postgres:11
    environment:
      - POSTGRES_USER=ara
      - POSTGRES_PASSWORD=ara
      - POSTGRES_DB=ara
```
