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
