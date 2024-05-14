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
docker create network frontend
docker create network backend
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
docker create volume base_portainer_data
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

### Files

* `files/base/nginx/html/favicon.ico`

### Templates

``` text title="templates/base/all/.env.j2"
ENV_TRAEFIK_HOST='{{ docker_deploy_base_traefik_host | mandatory }}'

ENV_TRAEFIK_EMAIL='{{ docker_deploy_base_traefik_email | default("dreknix@proton.me) }}'

ENV_TRAEFIK_REALM='{{ docker_deploy_base_traefik_realm | default("traefik") }}'

ENV_TRAEFIK_IPALLOWLIST='{{ docker_deploy_base_traefik_ipallowlist | default("127.0.0.1/8") }}'
```

* templates/base/all/.portainer_admin_secret.txt.j2
* templates/base/all/.portainer_key_secret.txt.j2

* templates/base/all/.traefik_secrets.txt.j2
* templates/base/all/traefik.env.j2
* templates/base/all/traefik-config/example.yaml.j2

* templates/base/all/nginx/html/index.html.j2

## License

[MIT](https://github.com/dreknix/docker-compose-base/blob/main/LICENSE)
