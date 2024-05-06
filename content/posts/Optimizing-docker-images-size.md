+++
title = 'Optimizing Docker Images Size'
date = 2024-05-06T15:27:12+02:00
draft = false
tags = ['docker']
+++

When you try to optimize the size of an image it is possible to display the size of every layers :
```bash
docker history <image-id>
```
For example : 
```bash
(ins)❯ docker history 7383c266ef25
IMAGE          CREATED       CREATED BY                                      SIZE      COMMENT
7383c266ef25   12 days ago   CMD ["nginx" "-g" "daemon off;"]                0B        buildkit.dockerfile.v0
<missing>      12 days ago   STOPSIGNAL SIGQUIT                              0B        buildkit.dockerfile.v0
<missing>      12 days ago   EXPOSE map[80/tcp:{}]                           0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENTRYPOINT ["/docker-entrypoint.sh"]            0B        buildkit.dockerfile.v0
<missing>      12 days ago   COPY 30-tune-worker-processes.sh /docker-ent…   4.62kB    buildkit.dockerfile.v0
<missing>      12 days ago   COPY 20-envsubst-on-templates.sh /docker-ent…   3.02kB    buildkit.dockerfile.v0
<missing>      12 days ago   COPY 15-local-resolvers.envsh /docker-entryp…   336B      buildkit.dockerfile.v0
<missing>      12 days ago   COPY 10-listen-on-ipv6-by-default.sh /docker…   2.12kB    buildkit.dockerfile.v0
<missing>      12 days ago   COPY docker-entrypoint.sh / # buildkit          1.62kB    buildkit.dockerfile.v0
<missing>      12 days ago   RUN /bin/sh -c set -x     && groupadd --syst…   113MB     buildkit.dockerfile.v0
<missing>      12 days ago   ENV PKG_RELEASE=1~bookworm                      0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENV NJS_RELEASE=2~bookworm                      0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENV NJS_VERSION=0.8.4                           0B        buildkit.dockerfile.v0
<missing>      12 days ago   ENV NGINX_VERSION=1.25.5                        0B        buildkit.dockerfile.v0
<missing>      12 days ago   LABEL maintainer=NGINX Docker Maintainers <d…   0B        buildkit.dockerfile.v0
<missing>      12 days ago   /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>      12 days ago   /bin/sh -c #(nop) ADD file:4b1be1de1a1e5aa60…   74.8MB
```
