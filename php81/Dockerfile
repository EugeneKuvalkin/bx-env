FROM composer AS composer

COPY ./composer.json /app/composer.json

RUN composer install --dev

FROM php:8.1-fpm

ARG APCU_VERSION=5.1.22

ARG CONTAINER_GID
ARG CONTAINER_UID
ARG CONTAINER_USER
ARG CONTAINER_GROUP

ARG BX_XDEBUG_IP
ARG BX_XDEBUG_PORT
ARG BX_DEFAULT_CHARSET

ARG BX_SMTP_PORT
ARG BX_SMTP_FROM

RUN apt-get update \
    && apt-get install -y vim zip unzip default-mysql-client git msmtp \
    && echo "syntax on" > ~/.vimrc \
    && apt-get -y install \
        libmagickwand-dev \
        libjpeg62-turbo-dev \
        libfreetype6-dev \
        memcached \
        libmemcached-dev \
        libicu-dev \
        libzip-dev \
        zlib1g-dev \
        libpng-dev \
        libxml2-dev \
    	libpq-dev \
    	librabbitmq-dev \
		libreoffice \
		libreoffice-java-common \
		default-jre \
		ffmpeg \
		procps \
        --no-install-recommends \
    && pecl install apcu-${APCU_VERSION} \
    && docker-php-ext-enable apcu \
    && pecl install memcached \
    && docker-php-ext-enable memcached \
    && pecl install igbinary \
    && docker-php-ext-enable igbinary \
    && docker-php-ext-configure gd --with-freetype=/usr/include/ --with-jpeg=/usr/include/ \
    && docker-php-ext-install -j$(nproc) gd \
    && docker-php-ext-install pgsql pdo_pgsql mysqli pdo_mysql opcache exif intl sockets bz2 zip \
    && docker-php-ext-configure opcache --enable-opcache \
    && rm -rf /tmp/* /var/cache/apk/* /var/lib/apt/lists/* \
    && pecl install xdebug && docker-php-ext-enable xdebug \
    && pecl install amqp && docker-php-ext-enable amqp \
    && docker-php-ext-configure pcntl --enable-pcntl && docker-php-ext-install pcntl

# install redis extension with igbinary support
RUN (echo "yes"; echo "no"; echo "no";) | pecl install redis-5.3.7 \
    && docker-php-ext-enable redis \
    # check if redis is installed
    && php -m | grep 'redis' \
    # check if redis is compiled with igbinary support
    && php -r 'echo Redis::SERIALIZER_IGBINARY;'

RUN cd /tmp && \
    git clone https://github.com/Imagick/imagick && \
    cd imagick && \
    phpize && \
    ./configure && \
    make && make install && \
    docker-php-ext-enable imagick && \
    cd /tmp && rm -rf imagick

RUN groupadd -g ${CONTAINER_GID} ${CONTAINER_GROUP} && \
    useradd -u ${CONTAINER_UID} -g ${CONTAINER_GID} ${CONTAINER_USER}

RUN chmod -R 777 /usr/local/etc/php/ \
    && chown -R ${CONTAINER_USER} /usr/local/etc/php/

COPY ./conf.d /usr/local/etc/php/conf.d/
COPY ./php-fpm.d /usr/local/etc/php-fpm.d/
COPY ./msmtp /usr/local/etc/msmtp/

RUN sed -i "$ a xdebug.client_host="${BX_XDEBUG_IP} /usr/local/etc/php/conf.d/xdebug.ini \
    && sed -i "$ a xdebug.client_port="${BX_XDEBUG_PORT} /usr/local/etc/php/conf.d/xdebug.ini \
    && sed -i "$ a default_charset="${BX_DEFAULT_CHARSET} /usr/local/etc/php/conf.d/php.ini \
    && sed -i "$ a mbstring.internal_encoding="${BX_DEFAULT_CHARSET} /usr/local/etc/php/conf.d/mbstring.ini \
    && sed -i "$ a port "${BX_SMTP_PORT} /usr/local/etc/msmtp/msmtp.conf \
    && sed -i "$ a from "${BX_SMTP_FROM} /usr/local/etc/msmtp/msmtp.conf

COPY --from=composer /usr/bin/composer /usr/bin/composer
COPY --from=composer /app /app

ENTRYPOINT php-fpm -F
