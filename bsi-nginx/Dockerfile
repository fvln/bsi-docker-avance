FROM debian:stretch-slim

RUN apt update && apt install -y nginx ssl-cert
COPY server.conf /etc/nginx/sites-available/server.conf
RUN ln -s /etc/nginx/sites-available/server.conf /etc/nginx/sites-enabled/server.conf
RUN rm /etc/nginx/sites-enabled/default

#RUN ln -s /dev/stdout /var/log/access.log
#RUN ln -s /dev/stderr /var/log/error.log

RUN echo '<?php phpinfo(); ?>' > /var/www/html/phpinfo.php

CMD nginx -g "daemon off; error_log stderr info;"
