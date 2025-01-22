# Docker compose instance for Traefik, Portainer, ...

Docker compose base instance (Traefik, Portainer, ...)

## Setup

Checkout the repository:

``` console
git clone https://github.com/dreknix/docker-compose-base.git base
cd base/
```

## Traefik

Adjust the configuration files:

The environment variables defined in `.env` configure the local adjustments of
this Docker compose instance.

``` console
cp .env.dist .env
nano .env
```

The following environment variables could be configured:

* `ENV_TRAEFIK_HOST` - fqdn of the Traefik host
* `ENV_TRAEFIK_EMAIL` - email for Let's Encrypt
* `ENV_TRAEFIK_REALM` - realm for basic authentication (default: `traefik`)
* `ENV_TRAEFIK_IPALLOWLIST` - ip allow list for Traefik (default: `127.0.0.1/8`)

The static configuration of Traefik should normally not be changed. The main
reason the adjust this configuration is to enable debug information.

``` console
cp traefik.env.dist traefik.env
nano traefik.env
```

Add users for the basic authentication of Traefik. The default user is `traefik`
with the same password.

``` console
cp .traefik_secrets.txt.dist .traefik_secrets.txt
nano .traefik_secrets.txt
```

Each line in this file is create via `htpasswd -nbB <user> "<pass>"`.

For additional dynamic configurations create a file in `traefik-config/`.

Create the networks:

``` console
docker network create frontend
docker network create backend
```

## Portainer

The password for the default `admin` user is set via the file
`.portainer_admin_secret.txt`. This password is stored in plain text in this
file.

``` console
nano .portainer_admin_secret.txt
```

For safety reasons the default `admin` user should be deleted after another user
with administrator permissions was created.

The portainer database is encrypted with the key stored in the file
`.portainer_key_secret.txt`.

``` console
nano .portainer_key_secret.txt
```

Create the volume:

``` console
docker volume create base_portainer_data
```

## Start

Start the first time in foreground to check the output.

``` console
docker compose up
```

## Test

Redirect your browser to `https://${ENV_TRAEFIK_HOST}/whoami` for testing.

The following services are included in this instance:

* `https://${ENV_TRAEFIK_HOST}/` - nginx web server for static files
* `https://${ENV_TRAEFIK_HOST}/traefik/`
* `https://${ENV_TRAEFIK_HOST}/portainer/`
* `https://${ENV_TRAEFIK_HOST}/whoami`

## Setup with Ansible

### Playbook

``` yaml
- name: Deploy Docker compose instance Traefik, Portainer, ...
  hosts: docker_base
  tasks:
    - name: Import role 'dreknix.docker_deploy'
      ansible.builtin.import_role:
        name: dreknix.docker_deploy
      vars:
        docker_deploy_name: base
        docker_deploy_git_repo: https://github.com/dreknix/docker-compose-base
        docker_deploy_file_dirs:
          - "{{ playbook_dir }}/files/base"
        docker_deploy_template_dirs:
          - "{{ playbook_dir }}/templates/base/all"
          - "{{ playbook_dir }}/templates/base/{{ inventory_hostname }}"
        docker_deploy_touched_files:
          - path: traefik-certs/acme.json
            mode: u=rw,go=
        docker_deploy_delete_unmanaged_files: true
        docker_deploy_volumes:
          - base_portainer_data
        docker_deploy_networks:
          - frontend
          - backend
```

### Variables

``` yaml
# fqdm of host
docker_deploy_base_traefik_host: dreknix.example.com

# list of Traefik users (generate passwords with: htpasswd -nbB user pass)
docker_deploy_base_traefik_users:
  - 'dreknix:$2y$05$P388kB5vG/I1Tv3qy8p.uOMeDgjh7ST54qS4RppuXjCQzJq/9U76C'

docker_deploy_base_traefik_api: true

docker_deploy_base_traefik_api_dashboard: true

docker_deploy_base_traefik_api_debug: false

docker_deploy_base_traefik_accesslog: false

# password for user 'admin' in Portainer
docker_deploy_base_portainer_admin_password: "**changeme**"

docker_deploy_base_portainer_key: "**changeme"
```

### Files

* `files/base/nginx/html/favicon.ico`

### Templates

#### Traefik

* `templates/base/all/.env.j2`

``` bash
ENV_TRAEFIK_HOST='{{ docker_deploy_base_traefik_host | mandatory }}'

ENV_TRAEFIK_EMAIL='{{ docker_deploy_base_traefik_email | default("dreknix@proton.me) }}'

ENV_TRAEFIK_REALM='{{ docker_deploy_base_traefik_realm | default("traefik") }}'

ENV_TRAEFIK_IPALLOWLIST='{{ docker_deploy_base_traefik_ipallowlist | default("127.0.0.1/8") }}'
```

* `templates/base/all/.traefik_secrets.txt.j2`

``` text
{%- for user in docker_deploy_base_traefik_users | default([]) %}
{{ user }}
{% endfor %}
```

* `templates/base/all/traefik.env.j2`

``` bash
#
# see: https://doc.traefik.io/traefik/reference/static-configuration/env/
#

# port 80 is needed for Let's Encrypt HTTP-01 challenge
TRAEFIK_ENTRYPOINTS_web="true"
TRAEFIK_ENTRYPOINTS_web_ADDRESS=":80"
# port 443 - main entry point
TRAEFIK_ENTRYPOINTS_websecure="true"
TRAEFIK_ENTRYPOINTS_websecure_ADDRESS=":443"

# add Docker provider - config via Docker labels
TRAEFIK_PROVIDERS_DOCKER="true"
# do not expose Docker labels of this instance
TRAEFIK_PROVIDERS_DOCKER_EXPOSEDBYDEFAULT="false"

# add File provider - config via files in /config
TRAEFIK_PROVIDERS_FILE_DIRECTORY="/config"
TRAEFIK_PROVIDERS_FILE_WATCH="true"

# if you omit this you will get the "Internal Server Error" due to
# self-signed certificate on an internal server port
TRAEFIK_SERVERSTRANSPORT_INSECURESKIPVERIFY="true"

# Certificates resolvers configuration must be part of the static
# configuration (here, environment variables)
TRAEFIK_CERTIFICATESRESOLVERS_LERESOLVER="true"
TRAEFIK_CERTIFICATESRESOLVERS_LERESOLVER_ACME_EMAIL="${ENV_TRAEFIK_EMAIL}"
TRAEFIK_CERTIFICATESRESOLVERS_LERESOLVER_ACME_STORAGE="/etc/letsencrypt/acme.json"
TRAEFIK_CERTIFICATESRESOLVERS_LERESOLVER_ACME_HTTPCHALLENGE="true"
TRAEFIK_CERTIFICATESRESOLVERS_LERESOLVER_ACME_HTTPCHALLENGE_ENTRYPOINT="web"
TRAEFIK_CERTIFICATESRESOLVERS_LERESOLVER_ACME_TLSCHALLENGE="false"

TRAEFIK_API="{{ docker_deploy_base_traefik_api | bool | lower}}"
TRAEFIK_API_DASHBOARD="{{ docker_deploy_base_traefik_api_dashboard | bool | lower }}"
TRAEFIK_API_DEBUG="{{ docker_deploy_base_traefik_api_debug | bool | lower }}"

TRAEFIK_ACCESSLOG="{{ docker_deploy_base_traefik_accesslog | bool | lower }}"
```

* `templates/base/all/traefik-config/example.yaml.j2`

#### Portainer

* `templates/base/all/.portainer_admin_secret.txt.j2`

``` text
{{ docker_deploy_base_portainer_admin_password }}
```

* `templates/base/all/.portainer_key_secret.txt.j2`

``` text
{{ docker_deploy_base_portainer_key }}
```

#### Static pages

* `templates/base/all/nginx/html/index.html.j2`

## License

[MIT](https://github.com/dreknix/docker-compose-base/blob/main/LICENSE)
