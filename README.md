# Docker compose instance for traefik, portainer, ...

Docker compose base instance (traefik, portainer, ...)

## Setup

Checkout the repository:

```console
$ git clone https://github.com/dreknix/docker-compose-base.git base
$ cd base/
```

Adjust the configuration files:

The environment variables defined in `.env` configure the local adjustments of
this Docker compose instance.

```console
$ cp .env.dist .env
$ nana .env
```

The following environment variables could be configured:

* `ENV_TRAEFIK_HOST` - fqdn of the traefik host
* `ENV_TRAEFIK_REALM` - realm for basic authentication (default: `traefik`)
* `ENV_TRAEFIK_WHITELIST` - ip whitelist for traefik (default: `127.0.0.1/8`)

The static configuration of traefik should normally not be changed. The main
reason the adjust this configuration is to enable debug information.

```console
$ cp traefik.env.dist traefik.env
$ nana traefik.env
```

Add users for the basic authentication of traefik. The default user is `traefik`
with the same password.

```console
$ cp .traefik_secrets.txt.dist .traefik_secrets.txt
$ nana .traefik_secrets.txt
```

Each line in this file is create via `htpasswd -nbB <user> "<pass>"`.

For additional dynamic configurations create a file in `traefik-config/`.

Create the networks:

```console
$ docker create network frontend
$ docker create network backend
```

## Start

Start the first time in foreground to check the output.

```console
$ docker compose up
```

## Test

Redirect your browser to `https://${ENV_TRAEFIK_HOST}/whoami` for testing.

The following services are included in this instance:

* `https://${ENV_TRAEFIK_HOST}/traefik/`
* `https://${ENV_TRAEFIK_HOST}/whoami`

## License

[MIT](https://github.com/dreknix/docker-compose-base/blob/main/LICENSE)
