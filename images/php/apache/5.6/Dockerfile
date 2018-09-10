FROM alpine:3.7
#
LABEL maintainer="Protopopys <protopopys@gmail.com>"
ENV LANG=en_US.UTF-8 \
    LANGUAGE=en_US.UTF-8 \
    LC_CTYPE=en_US.UTF-8 \
    LC_ALL=en_US.UTF-8
#
ENV HTTPD_PREFIX /usr/local/httpd
ENV PATH $HTTPD_PREFIX/bin:$PATH
#
ARG HTTPD_VERSION=2.4.34
ARG HTTPD_SHA256=fa53c95631febb08a9de41fd2864cfff815cf62d9306723ab0d4b8d7aa1638f0
ARG HTTPD_GPG_KEYS="A93D62ECC3C8EA12DB220EC934EA76E6791485A8 B9E8213AEFB861AF35A41F2C995E35221AD84DFF"
ENV HTTPD_PATCHES=""
#
ENV APACHE_DIST_URLS \
# https://issues.apache.org/jira/browse/INFRA-8753?focusedCommentId=14735394#comment-14735394
	https://www.apache.org/dyn/closer.cgi?action=download&filename= \
# if the version is outdated (or we're grabbing the .asc file), we might have to pull from the dist/archive :/
	https://www-us.apache.org/dist/ \
	https://www.apache.org/dist/ \
	https://archive.apache.org/dist/

ENV PATH $HTTPD_PREFIX/bin:$PATH
ENV APACHE_CONFDIR $HTTPD_PREFIX
#
WORKDIR $HTTPD_PREFIX
#
#Building httpd
#########################################
ENV runDeps \
		apr-dev \
		apr-util-dev \
		apr-util-ldap \
		perl
RUN set -x \
	&& addgroup -g 82 -S www-data \
	&& adduser -u 82 -D -S -G www-data www-data
RUN apk add --no-cache --virtual .httpd-build-deps \
		${runDeps} \
		ca-certificates \
		coreutils \
		dpkg-dev dpkg \
		gcc \
		gnupg \
		libc-dev \
		libressl \
		libressl-dev \
		libxml2-dev \
		lua-dev \
		make \
		nghttp2-dev \
		pcre-dev \
		tar \
		zlib-dev \
#		
	&& set -xe; \
	ddist() { \
		local f="$1"; shift; \
		local distFile="$1"; shift; \
		local success=; \
		local distUrl=; \
		for distUrl in $APACHE_DIST_URLS; do \
			if wget -O "$f" "$distUrl$distFile" && [ -s "$f" ]; then \
				success=1; \
				break; \
			fi; \
		done; \
		[ -n "$success" ]; }; \
	#
	ddist 'httpd.tar.bz2' "httpd/httpd-$HTTPD_VERSION.tar.bz2"; \
	echo "$HTTPD_SHA256 *httpd.tar.bz2" | sha256sum -c -; \
	#
	# see https://httpd.apache.org/download.cgi#verify
	ddist 'httpd.tar.bz2.asc' "httpd/httpd-$HTTPD_VERSION.tar.bz2.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in ${HTTPD_GPG_KEYS}; do \
		found=''; \
		for server in \
	    	ipv4.pool.sks-keyservers.net \
			ha.pool.sks-keyservers.net \
			hkp://keyserver.ubuntu.com:80 \
			hkp://p80.pool.sks-keyservers.net:80 \
			pgp.mit.edu \
		; do \
			echo "Fetching GPG key $key from $server"; \
			gpg --keyserver "${server}" --keyserver-options timeout=10 --recv-keys "$key" && found=yes && break; \
		done; \
		test -z "$found" && echo >&2 "error: failed to fetch GPG key $key" && exit 1; \
	done; \
	gpg --batch --verify httpd.tar.bz2.asc httpd.tar.bz2; \
	pkill -9 gpg-agent; \
    pkill -9 dirmngr; \
	rm -rf "$GNUPGHOME" httpd.tar.bz2.asc \
	&& mkdir -p src \
	&& tar -xf httpd.tar.bz2 -C src --strip-components=1 \
	&& rm httpd.tar.bz2 \
	&& cd src \
	&& set -xe; \
		patches() { \
		while [ "$#" -gt 0 ]; do \
			local patchFile="$1"; shift; \
			local patchSha256="$1"; shift; \
			ddist "$patchFile" "httpd/patches/apply_to_$HTTPD_VERSION/$patchFile"; \
			echo "$patchSha256 *$patchFile" | sha256sum -c -; \
			patch -p0 < "$patchFile"; \
			rm -f "$patchFile"; \
		done; } \
	&& patches $HTTPD_PATCHES \
	&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
	&& set -xe; \
		./configure \
		--build="$gnuArch" \
		--prefix="$HTTPD_PREFIX" \
		--enable-mods-shared=reallyall \
		--enable-mpms-shared=all \
	&& make -j "$(nproc)" \
	&& make install \
	&& cd .. \
	&& rm -r src man manual \
	&& sed -ri \
		-e 's!^(\s*CustomLog)\s+\S+!\1 /proc/self/fd/1!g' \
		-e 's!^(\s*ErrorLog)\s+\S+!\1 /proc/self/fd/2!g' \
		"$HTTPD_PREFIX/conf/httpd.conf" \
	&& httpdRunDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)" \
	&& apk add --virtual .httpd-rundeps $httpdRunDeps \
	&& apk del .httpd-build-deps
#Building PHP
#############################################
ARG GPG_KEYS="0BD78B5F97500D450838F95DFE857D9A90D90EC1 6E4F6AB321FDC07F2C332E3AC2BF0BC433CFC8B3"
ARG PHP_VERSION=5.6.36
ARG PHP_URL="https://secure.php.net/get/php-${PHP_VERSION}.tar.xz/from/this/mirror" 
ARG PHP_ASC_URL="https://secure.php.net/get/php-${PHP_VERSION}.tar.xz.asc/from/this/mirror"
ARG PHP_SHA256=18f536bf548e909b4e980379d0c4e56d024b2b1eb1c9768fd169360491f1d6dd
ARG PHP_MD5
ARG PHP_USER=www-data
ARG PHP_GROUP=www-data
ENV PHP_INI_DIR /usr/local/etc/php
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
ENV PHP_EXTRA_BUILD_DEPS apache2-dev
ENV PHP_EXTRA_CONFIGURE_ARGS --with-apxs2
#
COPY docker-php-* /usr/local/bin/
#
#Installing base packages
#
RUN	apk add --update --no-cache --virtual .persistent-deps \
	ca-certificates \
	curl \
	tar \
	xz \
	libressl \
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
		pkill -9 gpg-agent; \
    	pkill -9 dirmngr; \
		rm -rf "$GNUPGHOME"; \
	fi \
#
&& apk del .fetch-deps \
#
#Building PHP
#
&& apk add --update --no-cache --virtual .build-deps \
		$PHPIZE_DEPS \
		$PHP_EXTRA_BUILD_DEPS \
		coreutils \
		curl-dev \
		libedit-dev \
		libxml2-dev \
		sqlite-dev \
		libressl-dev \
&& export \
	CFLAGS="$PHP_CFLAGS" \
    CPPFLAGS="$PHP_CPPFLAGS" \
	LDFLAGS="$PHP_LDFLAGS" \
&& docker-php-source extract \
&& cd /usr/src/php \
&& gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
&& set -xe; \
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
&& rm -rf /tmp/pear ~/.pearrc
#
COPY httpd-foreground /usr/local/bin/
RUN chmod +x /usr/local/bin/httpd-foreground
WORKDIR /var/www/html
#
EXPOSE 80
CMD ["httpd-foreground"]