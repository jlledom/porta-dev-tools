# Porxy
Porxy is a porksy proxy of [porta](https://github.com/3scale/porta), a shitty solution to mediate the commnunication between [3scale/APIcast](https://github.com/3scale/APIcast), running in a container, and [3scale/porta](https://github.com/3scale/porta), running in local environment.

This is extrictly for development purposes and cannot be considered a replacement for Red Hat's official recommendations to work with 3scale.

## Requirements
Porxy assumes you can run 3scale/porta locally with whatever DBMS currently supported (MySQL, PostgreSQL, Oracle). See [installation instructions](https://github.com/3scale/porta/blob/master/INSTALL.md) for help.

You should also have Podman running in your environment.

A Redis instance is expected to be running locally and attending to port 6379, as well as a DNS resolver capable of handling wildcard domains, such as dnsmasq.

If you are using dnsmasq, make sure to include the following two DNS records:

```conf
address=/.3scale.localhost/127.0.0.1
address=/staging.apicast.dev/127.0.0.1
```

The instructions were tested in Mac OS X with zsh shell. You may addapt to your own environment accordingly.

#### Note: `host.docker.internal` on linux

On linux you can't use `host.docker.internal` for internal communication.
You should replace `host.docker.internal` with the IP of your container gateway.
In the default Docker configuration the IP address is `172.17.0.1`.
With rootless Podman default one is `10.0.2.2`.
You can find the IP address with one of the following commands:
```shell
ip -4 addr show docker0 2>/dev/null | grep -Po 'inet \K[\d.]+'
printf "%d.%d.%d.%d" $(awk '$2 == 00000000 && $7 == 00000000 { for (i = 8; i >= 2; i=i-2) { print "0x" substr($3, i-1, 2) } }' /proc/net/route)
```
This needs to be changed in the apisonator env file and `config/settings.yml` (`podman_internal_host: '172.17.0.1'`)

## Parameters

Consider the following general parameters for the [Setup](#setup) and [Run](#run) sections below. Other parameters may be indicated specifically for each component.

| Parameter | Description |
| ----------|-------------|
| `<APICAST_ACCESS_TOKEN>` | 3scale/APIcast special access token to access 3scale/Porta's Master API |
| `<PATH_TO_APISONATOR_ENV_FILE>` | Path to an env file to be used by 3scale/apisonator. |

## Setup

### Reset 3scale/porta databases
```shell
redis-cli flushall && bundle exec rails db:reset MASTER_PASSWORD=p USER_PASSWORD=p DEV_GTLD=local APICAST_ACCESS_TOKEN=<APICAST_ACCESS_TOKEN>
```

### Configure 3scale/porta domain substitution
```yaml
# config/domain_substitution.yml
default: &default
  enabled: false
  request_pattern: "\\.3scale\\.localhost"
  request_replacement: ".example.com"
  response_pattern: "\\.example\\.com"
  response_replacement: ".3scale.localhost"

development:
  <<: *default
  enabled: true

test:
  <<: *default

production:
  <<: *default

preview:
  <<: *default
```

### Env file for 3scale/apisonator
Prepare an env file for 3scale/apisonator:

```shell
echo "CONFIG_QUEUES_MASTER_NAME=host.containers.internal:6379/5\n\
CONFIG_REDIS_PROXY=host.containers.internal:6379/6\n\
CONFIG_INTERNAL_API_USER=system_app\n\
CONFIG_INTERNAL_API_PASSWORD=password\n\
RACK_ENV=production" > <PATH_TO_APISONATOR_ENV_FILE>
```

### Env file for 3scale/porta
Update an env file for 3scale/porta with following values:

```shell
CONFIG_INTERNAL_API_USER=system_app
CONFIG_INTERNAL_API_PASSWORD=password
```

## Run

The instructions below will run:
- [3scale/apisonator](https://github.com/3scale/apisonator) in a container (listerner attending to port 3001 and worker)
- [3scale/porta](https://github.com/3scale/porta) Rails server in local environment (at port 3000)
- [3scale/porta](https://github.com/3scale/porta) Sidekiq processes in local environment
- Porxy in a container (listening to port 3008)
- Staging [3scale/APIcast](https://github.com/3scale/APIcast) (listening to port 8080)

### Run 3scale/apisonator in a container

#### Listener
```
podman run -d --name apisonator --rm -p 3001:3001 --env-file <PATH_TO_APISONATOR_ENV_FILE> -it quay.io/3scale/apisonator:latest 3scale_backend start -p 3001 -l /var/log/backend/3scale_backend.log
```

#### Worker
```
podman run -d --name apisonator_worker --rm --env-file <PATH_TO_APISONATOR_ENV_FILE> -it quay.io/3scale/apisonator:latest 3scale_backend_worker run
```

### Run 3scale/porta locally

#### Porta Rails server
```
DEV_GTLD=local UNICORN_WORKERS=8 rails s -b 0.0.0.0
```

#### Porta Sidekiq (in another shell)
```
DEV_GTLD=local RAILS_MAX_THREADS=5 bundle exec rails sidekiq
```

### Run Porxy in a container

```
podman run -d --name porxy --rm -p 3008:3008 quay.io/guicassolato/porxy:latest
```

### Run 3scale/APIcast in a container

```
podman run -d --name apicast --rm -p 8080:8080 -e THREESCALE_PORTAL_ENDPOINT="http://<APICAST_ACCESS_TOKEN>@host.containers.internal:3008/master/api/proxy/configs" -e THREESCALE_DEPLOYMENT_ENV=staging -e BACKEND_ENDPOINT_OVERRIDE="http://host.containers.internal:3001" quay.io/3scale/apicast:master
```
