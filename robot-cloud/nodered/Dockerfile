FROM nodered/node-red:latest

USER root

RUN apk add --no-cache \
    libgpiod \
    libgpiod-dev \
    python3 \
    py3-pip \
    make \
    g++ \
    gcc

USER node-red

RUN npm install --unsafe-perm --omit=dev \
    @flowfuse/node-red-dashboard@^1.34.0 \
    node-red-node-pi-gpiod@^0.4.0 \
    node-red-node-pisrf@^0.4.0 \
    node-red-contrib-ds18b20-sensor@^1.3.6 \
    @thehighestbit/node-red-contrib-dht-libgpiod@^1.0.3