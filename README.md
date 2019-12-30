# Nextcloud Docker Compose

Docker compose file for a nextcloud server instance. It is based on this [docker image](https://hub.docker.com/_/nextcloud?tab=description).

The compose file is ment to run behind a reverse proxy that is available through the `my-reverse-proxy` network.

Remeber to use *more secure* environment variables than the defaults provided here, in this case:

- DB_PASSWORD: Password for your database
- DOCKER_STORAGE: Persistent path of your host for the docker image
- NEXTCLOUD_ADMIN_USER: Nextcloud admin user
- NEXTCLOUD_ADMIN_PASSWORD: Nextcloud admin password

Run it with:
```bash
docker-compose pull
docker-compose up -d --remove-orphans --build
```
## Example Setup

### nginx reverse proxy

In many cases, you run several docker images behind a reverse proxy. Here is how I use it:

Create a `docker network` to let services from different compose files communicate

```bash
docker network create my-reverse-proxy
build: build
```


Nginx `docker-compose` file:

```Dockerfile
version: '3.7'

services:
  nginx:
    container_name: nginx
    build: build
    restart: on-failure
    networks:
      - my-reverse-proxy
      - default
    ports:
      - 80:80
      - 443:443
    volumes:
      - ${TLS_CERTS_DIR:-./.test_certs}:/tls_certs/:ro
      - auth-nginx:/auth/

volumes:
  auth-nginx:

networks:
  my-reverse-proxy:
    name: my-reverse-proxy
```


... and the nextcloud `docker-compose` file:

```Dockerfile
version: '3.7'

services:
  nextcloud:
    container_name: nextcloud
    restart: always
    image: nextcloud:17-apache
    networks:
      - my-reverse-proxy
      - default
    expose:
      - 80
    volumes:
      - data-nextcloud:/var/www/html/
      - ${DOCKER_STORAGE}/nextcloud:/var/www/html/data
    depends_on:
      - nextcloud_db
      - nextcloud_redis
    environment:
      POSTGRES_HOST: nextcloud_db
      POSTGRES_USER: nextcloud
      POSTGRES_DB: nextcloud
      POSTGRES_PASSWORD: ${DB_PASSWORD:-test}
      NEXTCLOUD_TABLE_PREFIX: "nc_"
      NEXTCLOUD_ADMIN_USER: ${NEXTCLOUD_ADMIN_USER:-admin}
      NEXTCLOUD_ADMIN_PASSWORD: ${NEXTCLOUD_ADMIN_PASSWORD:-admin}
      REDIS_HOST: nextcloud_redis

  nextcloud_db:
    container_name: nextcloud_db
    image: postgres:12
    restart: always
    volumes:
      - data-db:/var/lib/postgresql
    environment:
      POSTGRES_USER: nextcloud
      POSTGRES_PASSWORD: ${DB_PASSWORD:-test}
      POSTGRES_DB: nextcloud

  nextcloud_redis:
    container_name: nextcloud_redis
    image: redis:alpine
    volumes:
        - data-redis:/data
    restart: always
    expose:
      - 6379

volumes:
  data-nextcloud:
  data-db:
  data-redis:

networks:
  my-reverse-proxy:
    external: true
```

In the nginx configuration, you need to set appropriate `proxy_pass` to your nextcloud instance.

Notes:
1. You can use the hostname `nextcloud` in the `proxy_pass` since they are sharing a network
2. In the example, `https_server.include` is a file providing my default https settings for nginx

It will look similar like the following:

```nginx
server {
    server_name nextcloud.example.com;
    include conf.d/common/https_server.include;
    client_max_body_size       10G;
    proxy_request_buffering    off;

     location / {
         proxy_pass http://nextcloud:80/;
         include conf.d/common/proxy.include;
     }
}
```
