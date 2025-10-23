本仓库的制作初衷是由于``lejianwen``于``20251022``不支持``多RELAY``，而我正好有阿里云的一台服务器CDT用完了，想着换一个中继不太好用，在ISSUEs发现了``wy41401``作者写的多RELAY，于是结合了一下

本仓库从官方仓库拉取镜像，结合lejianwen的：解决了使用API链接超时问题，可以强制登录后才能发起链接，支持客户端websocket，和wy41401的多地RELAY中继服务

|参考人员|仓库名|仓库链接|
|---|---|---|
|lejianwen|rustdesk-server|https://github.com/lejianwen/rustdesk-server|
|wy414012|rustdesk-server|https://github.com/wy414012/rustdesk-server|

中国可以用``ghcr.nju.edu.cn``

```
version: '3'

networks:
  rustdesk-net:
    external: false

services:
  hbbs:
    container_name: hbbs
    ports:
      - 21115:21115
      - 21116:21116
      - 21116:21116/udp
      - 21118:21118
    image: ghcr.io/arkylin/rustdesk-server:latest
    command: hbbs -r <server[:21117]>
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    depends_on:
      - hbbr
    restart: unless-stopped

  hbbr:
    container_name: hbbr
    ports:
      - 21117:21117
      - 21119:21119
    image: ghcr.io/arkylin/rustdesk-server:latest
    command: hbbr
    volumes:
      - ./data:/root
    networks:
      - rustdesk-net
    restart: unless-stopped
  api:
    container_name: api
    ports:
      - 21114:21114
    image: lejianwen/rustdesk-api
    environment:
      - TZ=Asia/Shanghai
      - RUSTDESK_API_RUSTDESK_ID_SERVER=<server[:21116]> #21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=<server[:21117]> #21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=http://<server[:21114]> #21114
      #- RUSTDESK_API_RUSTDESK_KEY=<key>
    volumes:
      - /data/rustdesk/api:/app/data #将数据库挂载
      - /data/rustdesk/server:/app/conf/data #挂载key文件到api容器，可以不用使用 RUSTDESK_API_RUSTDESK_KEY
    networks:
      - rustdesk-net
    restart: unless-stopped
```
