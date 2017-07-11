FROM ruby:2.3.1-alpine

MAINTAINER KevinGuo <chinakevinguo@live.com>

ENV TZ 'Asia/Shanghai'

RUN apk upgrade --no-cache && \
    apk add --no-cache bash tzdata nodejs py-pygments git gcc musl-dev make && \
    apt-get install ruby-dev && \
    git clone https://github.com/chinakevinguo/chinakevinguo.github.io.git /root/chinakevinguo.github.io && \
    cd /root/chinakevinguo.github.io && \
    gem install --no-document bundle && \
    bundle install && \
    ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone && \
    apk del gcc musl-dev make && \
    rm -rf /var/cache/apk/*

ADD cron/15min/* /etc/periodic/15min

ADD entrypoint.sh /root/entrypoint.sh

WORKDIR /root

CMD ["/root/entrypoint.sh"]
