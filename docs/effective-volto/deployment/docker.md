---
myst:
  html_meta:
    "description": "Docker"
    "property=og:description": "Docker"
    "property=og:title": "Docker"
    "keywords": "Volto, Plone, Docker"
---

# Docker

As any application on earth, Plone 6 Volto can be deployed using Docker.
It is recommended that you follow the best practices explained in the "Installing Plone" recent training course by Érico Andréi and Jens Klein.

## Simple example

You'll find in the Volto repository a simple `docker-compose` example:

https://github.com/plone/volto/blob/master/docker-compose.yml

```shell
docker-compose -f <following_snippet.yml> up
```

```yaml
version: '3.3'
services:

  backend:
    image: plone/plone-backend:6.0.0b2
    # Plone 5.2 series can be used too
    # image: plone/plone-backend:5.2.9
    ports:
      - '8080:8080'
    environment:
      - SITE=Plone
      - 'ADDONS=plone.restapi==8.29.0 plone.volto==4.0.0a13 plone.rest==2.0.0a5'
      - 'PROFILES=plone.volto:default-homepage'

  frontend:
    image: 'plone/plone-frontend:latest'
    ports:
      - '3000:3000'
    restart: always
    environment:
      # These are needed in a Docker environment since the
      # routing needs to be amended. You can point to the
      # internal network alias.
      RAZZLE_INTERNAL_API_PATH: http://backend:8080/Plone
      RAZZLE_DEV_PROXY_API_PATH: http://backend:8080/Plone
    depends_on:
      - backend
```

## Full with reverse proxy

```shell
docker-compose -f <following_snippet.yml> up
```

```yaml
version: '3.3'
services:

  backend:
    image: plone/plone-backend:6.0.0b2
    # Plone 5.2 series can be used too
    # image: plone/plone-backend:5.2.9
    ports:
      - '8080:8080'
    environment:
      - SITE=Plone
      - 'ADDONS=plone.restapi==8.29.0 plone.volto==4.0.0a13 plone.rest==2.0.0a5'
      - 'PROFILES=plone.volto:default-homepage'
    labels:
      - traefik.enable=true
      # SERVICE
      - traefik.http.services.plone-backend.loadbalancer.server.port=8080
      # Plone API
      - traefik.http.routers.backend.rule=Host(`plone.localhost`) && PathPrefix(`/++api++`)
      - traefik.http.routers.backend.service=plone-backend
      - "traefik.http.middlewares.backend.replacepathregex.regex=^/\\+\\+api\\+\\+($$|/.*)"
      - "traefik.http.middlewares.backend.replacepathregex.replacement=/VirtualHostBase/http/plone.localhost/Plone/++api++/VirtualHostRoot/$$1"
      - traefik.http.routers.backend.middlewares=gzip,backend

  frontend:
    image: 'plone/plone-frontend:latest'
    ports:
      - '3000:3000'
    restart: always
    environment:
      # These are needed in a Docker environment since the
      # routing needs to be amended. You can point to the
      # internal network alias.
      RAZZLE_INTERNAL_API_PATH: http://backend:8080/Plone
      # This last is not required if a reverse proxy is in the stack
      RAZZLE_DEV_PROXY_API_PATH: http://backend:8080/Plone
    depends_on:
      - backend
    labels:
      - traefik.enable=true
      # SERVICE
      - traefik.http.services.plone-frontend.loadbalancer.server.port=3000
      # HOSTS: Main
      - traefik.http.routers.frontend.rule=Host(`plone.localhost`)
      - traefik.http.routers.frontend.service=plone-frontend
      - traefik.http.routers.frontend.middlewares=gzip

  reverse-proxy:
    # The official v2 Traefik docker image
    image: traefik:v2.8
    # Enables the web UI and tells Traefik to listen to docker
    command: --api.insecure=true --providers.docker
    ports:
      # The HTTP port
      - "80:80"
      # The Web UI (enabled by --api.insecure=true)
      - "8888:8080"
    volumes:
      # So that Traefik can listen to the Docker events
      - /var/run/docker.sock:/var/run/docker.sock
    labels:
      - traefik.http.middlewares.gzip.compress=true
      - traefik.http.middlewares.gzip.compress.excludedcontenttypes=image/png, image/jpeg, font/woff2
```

## Recommended setup

In a production environment, independently of the orchestrator of your choice the Plone AI-Team recommends the practices in the project structure resultant of running this generator:

https://github.com/collective/cookiecutter-plone-starter

As we've seen at: {ref}`bootstrapping-label`.

Take a look at the `devops` folder, you'll find all the relevant information there.
