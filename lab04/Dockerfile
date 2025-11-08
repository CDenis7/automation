FROM jenkins/ssh-agent

RUN apt-get update && apt-get install -y php-cli wget php-curl php-xml php-mbstring unzip php-zip

RUN wget https://getcomposer.org/installer -O /tmp/composer-setup.php
RUN php /tmp/composer-setup.php --install-dir=/usr/local/bin --filename=composer
RUN rm /tmp/composer-setup.php