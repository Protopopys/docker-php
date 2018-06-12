FROM alpine:3.7

LABEL maintainer="Protopopys <protopopys@gmail.com>"
#
ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8 \
    LC_CTYPE=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8
#For PHP 5.3
ENV	OPENSSL_VERSION="1.0.2o"
#	
ARG GPG_KEYS
ARG PHP_VERSION
ARG PHP_URL="https://secure.php.net/get/php-${PHP_VERSION}.tar.xz/from/this/mirror" 
ARG PHP_ASC_URL="https://secure.php.net/get/php-${PHP_VERSION}.tar.xz.asc/from/this/mirror"
ARG PHP_SHA256 
ARG PHP_MD5
ARG PHP_USER=daemon
ARG PHP_GROUP=daemon
ENV PHP_INI_DIR /usr/local/etc/php
ENV PHP_EXTRA_CONFIGURE_ARGS --enable-fpm --with-fpm-user=${PHP_USER} --with-fpm-group=${PHP_GROUP} --disable-cgi
ENV PHP_CFLAGS="-fstack-protector-strong -fpic -fpie -O2"
ENV PHP_CPPFLAGS="$PHP_CFLAGS"
ENV PHP_LDFLAGS="-Wl,-O1 -Wl,--hash-style=both -pie"
ENV PHPIZE_DEPS \
		autoconf \
		dpkg-dev dpkg \
		file \
		g++ \
		gcc \
		libc-dev \
		make \
		pkgconf \
		re2c
COPY docker-php-* /usr/local/bin/
#
#Installing base packages
#
RUN set -xe; \
	ssl_package=''; \
	#
	if [ "$(echo ${PHP_VERSION} | cut -f 1 -d ".")" -ge 5 ] && [ "$(echo ${PHP_VERSION} | cut -f 2 -d ".")" -ge 6 ] ; then \
		ssl_package='libressl'; \
	fi; \
	#
	apk add --update --no-cache --virtual .persistent-deps \
	ca-certificates \
	curl \
	tar \
	xz \
	${ssl_package} \
&& mkdir -p $PHP_INI_DIR/conf.d \
#
#Checking php package.
#
&& set -xe; \
	apk add --update --no-cache --virtual .fetch-deps \
		wget \
		gnupg \
	; \
	mkdir -p /usr/src; \
	cd /usr/src; \
	wget -O php.tar.xz "$PHP_URL"; \
    #
	if [ -n "$PHP_SHA256" ]; then \
		echo "$PHP_SHA256 *php.tar.xz" | sha256sum -c -; \
	fi; \
    #
	if [ -n "$PHP_MD5" ]; then \
		echo "$PHP_MD5 *php.tar.xz" | md5sum -c -; \
	fi; \
    #
	if [ -n "$PHP_ASC_URL" ]; then \
		wget -O php.tar.xz.asc "$PHP_ASC_URL"; \
		export GNUPGHOME="$(mktemp -d)"; \
		for key in ${GPG_KEYS}; do \
			found=''; \
			for server in \
	    		ipv4.pool.sks-keyservers.net \
				ha.pool.sks-keyservers.net \
				hkp://keyserver.ubuntu.com:80 \
				hkp://p80.pool.sks-keyservers.net:80 \
				pgp.mit.edu \
			; do \
				echo "Fetching GPG key $GPG_KEYS from $server"; \
				gpg --keyserver "${server}" --keyserver-options timeout=10 --recv-keys "$key" && found=yes && break; \
			done; \
			test -z "$found" && echo >&2 "error: failed to fetch GPG key $key" && exit 1; \
		done; \
		gpg --batch --verify php.tar.xz.asc php.tar.xz; \
		rm -rf "$GNUPGHOME"; \
	fi \
#
&& apk del .fetch-deps \
#
#Building PHP
#
&& set -xe; \
	extra_package=''; \
	if [ "$(echo ${PHP_VERSION} | cut -f 1 -d ".")" -ge 7 ] && [ "$(echo ${PHP_VERSION} | cut -f 2 -d ".")" -ge 2 ] ; then \
		extra_package='libsodium-dev'; \
	fi ; \
	ssl_packages=''; \
	if [ "$(echo ${PHP_VERSION} | cut -f 1 -d ".")" -eq 5 ] && [ "$(echo ${PHP_VERSION} | cut -f 2 -d ".")" -lt 6 ] ; then \
		ssl_packages=''; \
	else \
		ssl_packages='libressl-dev'; \
	fi; \
 	apk add --update --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		coreutils \
		curl-dev \
		libedit-dev \
		libxml2-dev \
		sqlite-dev \
		${ssl_packages} \
		${extra_package} \
&& set -xe; \
	if [ "$(echo ${PHP_VERSION} | cut -f 1 -d ".")" -eq 5 ] && [ "$(echo ${PHP_VERSION} | cut -f 2 -d ".")" -lt 6 ] ; then \
    	cd /tmp; \
      	mkdir openssl; \
      	curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz" -o openssl.tar.gz; \
      	curl -sL "https://www.openssl.org/source/openssl-$OPENSSL_VERSION.tar.gz.asc" -o openssl.tar.gz.asc; \
      	#gpg --verify openssl.tar.gz.asc; \
      	tar -xzf openssl.tar.gz -C openssl --strip-components=1; \
      	cd /tmp/openssl; \
      	./config && make && make install; \
      	rm -rf /tmp/*; \
	fi \
&& export \
	CFLAGS="$PHP_CFLAGS" \
    CPPFLAGS="$PHP_CPPFLAGS" \
	LDFLAGS="$PHP_LDFLAGS" \
&& docker-php-source extract \
&& cd /usr/src/php \
&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
&& set -xe; \
	extra_module=''; \
	if [ "$(echo ${PHP_VERSION} | cut -f 1 -d ".")" -ge 7 ] && [ "$(echo ${PHP_VERSION} | cut -f 2 -d ".")" -ge 2 ] ; then \
		extra_module='--with-sodium=shared'; \
	fi ; \
	./configure \
	--build="$gnuArch" \
	--with-config-file-path="$PHP_INI_DIR" \
	--with-config-file-scan-dir="$PHP_INI_DIR/conf.d" \
	--enable-ftp \
	--enable-mbstring \
	--enable-mysqlnd \
	--with-curl \
	--with-libedit \
	--with-openssl \
	--with-zlib \
	${extra_module} \
	$(test "$gnuArch" = 's390x-linux-gnu' && echo '--without-pcre-jit') \
	$PHP_EXTRA_CONFIGURE_ARGS \
&& make -j "$(nproc)" \
&& make install \
&& { find /usr/local/bin /usr/local/sbin -type f -perm +0111 -exec strip --strip-all '{}' + || true; } \
&& make clean \
&& cd / \
&& docker-php-source delete \
&& runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)" \
&& apk add --no-cache --virtual .php-rundeps $runDeps \
&& apk del .build-deps \
#
#EnvSubst
#
&& apk add --no-cache --virtual .gettext gettext \
&& mv /usr/bin/envsubst /tmp/ \
&& envsubstDeps="$( \
	scanelf --needed --nobanner --format '%n#p' /tmp/envsubst \
	| tr ',' '\n' \
	| sort -u \
	| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)" \
&& apk add --no-cache tzdata \
&& apk add --no-cache --virtual .envsubst-rundeps $envsubstDeps \	
&& apk del .gettext \
&& mv /tmp/envsubst /usr/local/bin/ \
&& rm -rf /tmp/pear ~/.pearrc \
&& rm -r /usr/src/* \
&& set -ex; \
	if [ "$(echo ${PHP_VERSION} | cut -f 1 -d ".")" -ge 7 ] && [ "$(echo ${PHP_VERSION} | cut -f 2 -d ".")" -ge 2 ] ; then \
		docker-php-ext-enable sodium; \
	fi; \
#
#Preparing php-fpm
#
	cd /usr/local/etc; \
	if [ -d php-fpm.d ]; then \
		sed 's!=NONE/!=!g' php-fpm.conf.default | tee php-fpm.conf > /dev/null; \
		cp php-fpm.d/www.conf.default php-fpm.d/www.conf; \
	else \
		mkdir php-fpm.d; \
		cp php-fpm.conf.default php-fpm.d/www.conf; \
		{ \
			echo '[global]'; \
			echo 'include=etc/php-fpm.d/*.conf'; \
		} | tee php-fpm.conf; \
	fi; \
#
	{ \
		echo '[global]'; \
		echo 'error_log = /proc/self/fd/2'; \
		echo; \
		echo '[www]'; \
		echo '; if we send this to /proc/self/fd/1, it never appears'; \
		echo 'access.log = /proc/self/fd/2'; \
		echo; \
#		echo 'clear_env = no'; \
		echo; \
		echo '; Ensure worker stdout and stderr are sent to the main error log.'; \
		echo 'catch_workers_output = yes'; \
	} | tee php-fpm.d/docker.conf; \
#	
	{ \
		echo '[global]'; \
		echo 'daemonize = no'; \
		echo; \
		echo '[www]'; \
		echo 'listen = 9000'; \
	} | tee php-fpm.d/zz-docker.conf

ENTRYPOINT ["docker-php-entrypoint"]

EXPOSE 9000
CMD ["php-fpm"]