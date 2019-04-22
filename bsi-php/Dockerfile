FROM debian:stretch-slim

RUN apt update && apt install -y php-fpm php-mysql
RUN sed -i -e 's#listen = /run/php/php7.0-fpm.sock#listen = 9000#' /etc/php/7.0/fpm/pool.d/www.conf
RUN echo 'catch_workers_output = yes' >> /etc/php/7.0/fpm/pool.d/www.conf

RUN mkdir /run/php

CMD /usr/sbin/php-fpm7.0 --nodaemonize

