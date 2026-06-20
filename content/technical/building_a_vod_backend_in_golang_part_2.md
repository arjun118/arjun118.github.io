+++
date = '2026-06-18T23:54:25+05:30'
draft = false
title = 'Building a Vod Backend in Golang Part 2'
tags=['technical','golang']
+++

we will be introducing nginx now, as a reverse proxy (acts on behalf of servers to field incoming client requests securely)

imma keep the name of the approaches simple

1. Nginx Reverse Proxy
2. Nginx Reverse Proxy With Cache
3. X-Accel-Redirect

we will be seeing an extra file, `nginx.conf` and some code changes in the application server.

Let's start then

# Implementations

## Nginx Reverse Proxy (`branch: nginx_reverse_proxy`)

The Provider and the Delivery Will still remain with our minio-instance.

If you remember in our `Public Minio storage and delivery (branch: public_minio_hls)`, we directly access the video from the minio's playbackURL. eg: `http://minio:9000/storage/video/...../object_key.m3u8`

> Our storage bucket will be `public` in this impl

It will be the same here, but instead the client makes a request to the nginx URI path. Nginx will fetch the video requested resource for you from our minio-instance.

lets take a look at the nginx config `nginx.conf`

### Nginx Configuration file

in: `nginx/nginx.conf`

```nginx
events {}

http {
    server{
        listen 80;
        client_max_body_size 500M;
        location /api/videos {
            proxy_pass http://backend:8080;
            proxy_set_header HOST $host;
            proxy_set_header X-Real-IP $remote_addr;

            proxy_read_timeout 600s;
            proxy_send_timeout 600s;
        }

        location /media/ {
            proxy_pass http://minio:9000/;
        }
    }
}

```

### Docker Compose file

```dockerfile
services:
  minio:
    image: quay.io/minio/minio
    container_name: minio
    restart: always
    # Expose these ports during local development/debugging.
    # In a typical production deployment MinIO lives on a private network
    # and is accessed only by trusted services (Nginx, backend, workers, etc).
    # The MinIO Console (9001) is almost never exposed publicly.
    # The S3 API port (9000) may be exposed if MinIO is being used as the
    # public delivery layer, but is commonly hidden behind Nginx, a CDN,
    # or another gateway. [comment- written by AI]
    # ports:
      # - "9000:9000"
      # - "9001:9001"
    environment:
      MINIO_ROOT_USER: adminpass
      MINIO_ROOT_PASSWORD: adminpass
    command: server --console-address ":9001" /data
    volumes:
      - ~/minio-data:/data
  nginx:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - backend
      - minio
  backend:
    build:
      context: .
      dockerfile: internal/Dockerfile
    depends_on:
      - minio
```

1. any request that matches the uri path - `/api/videos` will be proxied by nginx to our backend - without stripping the matched prefix:

```
1. Request url by client: POST `http://localhost:8080/api/videos/.....`

2. Nginx matches the incoming request URI with `/api/videos/` and proxies to our backend on behalf of the client

3. Request made to our application server by nginx: POST `http://backend:8080/api/videos/....`
```

2. any request that matches the uri path - `/media/` will be proxied by nginx to our minio-instance- by stripping the matched prefix

### Request Lifecycle

![nginx reverse proxy flow](/images/nginx_reverse_proxy.png)

1. video bytes never touches our application server.

### Working

![nginx reverse proxy demo](/images/nginx_reverse_proxy_demo.png)

1. run: `docker compose up --build`
2. `curl -H "Authorization:Bearer secret" -X POST http://localhost:8080/api/videos -F "file=@/path/to/you/video.mp4"`
3. i have implemented a simple middle to mimic authentication which i will cover in the last part when we get to cookie for video-access.

```
notice the server as `nginx:1.31.2` from our response headers in for the request to `playlist0.ts` - reverse proxy in action
```

## Nginx Reverse Proxy Cache (`branch: nginx_reverse_proxy_cache`)

1. `docker files`, `provider`, `delivery` - all else remain the same except for `nginx.conf`

The only change for this implementation will be with the file `nginx/nginx.conf`

here is the configuration file

```nginx
events {}


http {
    log_format cachelog
    '$remote_addr '
    '$request_uri '
    '$status '
    '$upstream_cache_status';

    access_log /var/log/nginx/cache.log cachelog;


    proxy_cache_path /var/cache/nginx/hls_cache
        levels=1:2
        keys_zone=hls_cache:20m
        max_size=50g
        inactive=24h
        use_temp_path=off;


    server{
        listen 80;
        client_max_body_size 500M;

        location / {
            proxy_pass http://backend:8080/;
        }

        location /api/videos {

            proxy_pass http://backend:8080;
            proxy_set_header HOST $host;
            proxy_set_header X-Real-IP $remote_addr;

            proxy_read_timeout 600s;
            proxy_send_timeout 600s;

        }


        location /media/ {

            proxy_cache hls_cache;

            proxy_cache_key "$scheme$proxy_host$request_uri";

            proxy_ignore_headers Cache-Control Expires Set-Cookie;

            proxy_cache_valid 200 24h;
            proxy_cache_valid 206 24h;

            proxy_cache_use_stale
                error
                timeout
                http_500
                http_502
                http_503
                http_504;

            proxy_cache_background_update on;
            proxy_cache_lock on;
            proxy_cache_lock_timeout 10s;

            proxy_force_ranges on;

            add_header X-Cache-Status $upstream_cache_status always;

            proxy_pass http://minio:9000/;
        }

    }
}
```

### Request Lifecycle

![nginx porxy cache flow](/images/nginx_reverse_proxy_cache_flow.png)

1. when the requested object/resource results in a cache miss , nginx will proxy the request to minio-instance and fetch from the source (minio)
2. on proxying response to the client nginx caches the response data.
3. in the diagram , in `delivery` the `green lines` show a `cache hit` and `red lines` show a `cache miss`

### Working

![nginx proxy cache demo](/images/nginx_reverse_proxy_cache_demo.png)

1. observe the first request log - it's a cache `MISS`
2. now when i refreshed the page, essentially requesting the same resource again (manifest and video segements) - it's a cache `HIT`. hence proxy cache in action

## Closing

1. lastly we will cover the `X-Accel-Redirect` along with scoped authentication for application server api routes and Cookie based authentication to access video data.
