+++
title = 'Local Gitlab with HTTP Docker repo'
date = 2023-12-03T22:34:37-04:00
+++

Soooo, the router I was using to bypass my modem to make my homelab public died. And I really need to learn more about GitLab for a conference presentation I will be doing in early '24. So I thought to myself "I'll just spin up a docker instance of gitlab. It has all the features I need".

And it does; git, pipelines, docker repo, but the changes over the years to the underlying api's have left many of the community documentation pretty fragemented. Or maybe my google-fu failed me. Either way, no solution I found worked so here are the steps I had to take for my particular setup :

![Homelab structure](./gitlab.png)

At the core of my homelab is Proxmox which at the moment is split into two VMs, a big one and a much much smaller one. The big one runs my Portainer instance which I use to manage all my in home services and thats where I deployed Gitlab-CE [via docker-compose](#gitlab-docker-compose). This file has a envvar `GITLAB_OMNIBUS_CONFIG` that is used to confure sections of GitLab. Here is where you will add your local IP, enable the docker repo, set your port if needed, etc

The small vm is my gitlab runner. This runner is configured as a docker executor. The goal here is to be able to run DinD builds so that we are can publish into the GitLab Docker Repo.

But remember this unforunately is all occuring behind my crappy HomeHub3000 modem that (I believe intentionally) can't port forward, so no SSL, so most of the default examples don't work in my case.

After many different variations of which machine had which files, it really boils down to three sets of steps:

## Gitlab Instance

The host of your Gitlab instance, in my case the VM hosting my Portainer, edit/create `/etc/docker/daemon.json` and add the following:

```json
{ "insecure-registries": ["192.168.1.31:5001"] }
```

Then restart your daemon and docker serivce. This varies by system but own my Debian setup,

```sh
systemctl daemon-reload &&  systemctl restart docker
```

## Runner Instance

On my runner I had to first compile [docker-credential-pass](https://github.com/docker/docker-credential-helpers/) and add it to my path.

Then create a new gpg2 key with `gpg2 --gen-key` which should produce something like this after answering all of it's questions; `pub 0E1A0E5A636693CDE41C1E6B5FC870A52E362877`.

Copy the value and init pass with it like this `pass init 0E1A0E5A636693CDE41C1E6B5FC870A52E362877`.

After that edit/create `~/.docker/config.json` to look like this:

```json
{
  "credsStore": "pass"
}
```

Still on the runner vm edit the runner config found at `/etc/gitlab-runner/config.toml` to disable tls and enable privileged mode. Below is my whole runner config.toml

```toml
concurrent = 1
check_interval = 0
shutdown_timeout = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "debian-docker"
  url = "http://192.168.1.31"
  id = 2
  token = "****************"
  token_obtained_at = 2023-12-02T12:22:41Z
  token_expires_at = 0001-01-01T00:00:00Z
  executor = "docker"
  [runners.cache]
    MaxUploadedArchiveSize = 0
  [runners.docker]
    tls_verify = false
    image = "docker:latest"
    privileged = true
    disable_cache = false
    volumes = ["/cache"]

```

## .gitlab-ci.yml

In your `gitlab-cy.yml` file, since we're using DinD we need to pass into the service command some flags to disable tls and enable privileged mode. You'll also notice that I had set my envvars for DOCKER_TLS_CERTDIR to an empty string and disable buildkit. This is particularly frustarting as it seems to be a [sovled issue](https://github.com/moby/moby/pull/45992) but I was too tired at this point to fight anymore.

Also, note the login. `--password-stdin` with buildkit disabled seems to fail to log in properly, forcing me to login in such a manner.

```json
stages:
  - build
  - deploy

variables:
  DOCKER_TLS_CERTDIR: ""
  DOCKER_BUILDKIT: 0

compile:
  tags:
    - linux
  image: docker:24.0.7

  services:
    - name: docker:24.0.7-dind
      command: ["--tls=false", "--insecure-registry=192.168.1.31:5001"]
  stage: build
  before_script:
    - apk add git
  script:
    - docker login -u $USER -p $PASS 192.168.1.31:5001
    - docker build -t 192.168.1.31:5001/forms/server/api .
    - docker push 192.168.1.31:5001/forms/server/api
```

## Gitlab Docker-Compose

```yaml
version: "3.6"
services:
  web:
    image: "gitlab/gitlab-ce:16.6.1-ce.0"
    restart: always
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        gitlab_rails['lfs_enabled'] = true;
        external_url 'http://192.168.1.31'
        registry_external_url 'http://192.168.1.31:5001'
        gitlab_rails['registry_enabled'] = true
        gitlab_rails['registry_port'] = "5001"
        gitlab_rails['registry_path'] = "/var/opt/gitlab/gitlab-rails/shared/registry"
        registry['env'] = {
          "REGISTRY_HTTP_RELATIVEURLS" => true
        }

    ports:
      - "80:80"
      - "443:443"
      - "23:22"
      - "5001:5001"
    volumes:
      - "/srv/gitlab/config:/etc/gitlab"
      - "/srv/gitlab/logs:/var/log/gitlab"
      - "/srv/gitlab/data:/var/opt/gitlab"
    shm_size: "256m"
```

That's pretty much it. Hopefully this will save someone some time in the future!
