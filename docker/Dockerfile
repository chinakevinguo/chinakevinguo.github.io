FROM nginx:1.11.10-alpine

MAINTAINER KevinGuo "chinakevinguo@live.com"

ENV TZ 'Asia/Shanghai'

RUN apk upgrade --no-cache \
    && apk add --no-cache bash git \
    && rm -rf /usr/share/nginx/html \
    && git clone https://github.com/chinakevinguo/kevinguo.me.git /usr/share/nginx/html \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && rm -rf /var/cache/apk/*

ADD entrypoint.sh /entrypoint.sh

WORKDIR /usr/share/nginx/html

CMD ["/entrypoint.sh"]
