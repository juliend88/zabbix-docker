FROM alpine:latest
MAINTAINER Alexey Pustovalov <alexey.pustovalov@zabbix.com>

ARG APK_FLAGS_COMMON="-q"
ARG APK_FLAGS_PERSISTANT="${APK_FLAGS_COMMON} --clean-protected --no-cache"
ARG APK_FLAGS_DEV="${APK_FLAGS_COMMON} --no-cache"
ARG DB_TYPE=sqlite3
ENV TERM=xterm
ENV MIBDIRS=/var/lib/zabbix/mibs MIBS=+ALL

RUN addgroup zabbix && \
    adduser -S \
            -D -G zabbix \
            -h /var/lib/zabbix/ \
        zabbix && \
    mkdir -p /etc/zabbix/ && \
    mkdir -p /var/lib/zabbix && \
    mkdir -p /var/lib/zabbix/enc && \
    mkdir -p /var/lib/zabbix/modules && \
    mkdir -p /var/lib/zabbix/ssh_keys && \
    mkdir -p /var/lib/zabbix/ssl && \
    mkdir -p /var/lib/zabbix/ssl/certs && \
    mkdir -p /var/lib/zabbix/ssl/keys && \
    mkdir -p /var/lib/zabbix/ssl/ssl_ca && \
    chown --quiet -R zabbix:root /var/lib/zabbix && \
    mkdir -p /usr/lib/zabbix/externalscripts && \
    apk update && \
    apk add ${APK_FLAGS_PERSISTANT} \
            supervisor \
            bash \
            sqlite-libs \
            fping \
            libxml2 \
            unixodbc \
            net-snmp-agent-libs \
            libldap \
            libcurl \
            openipmi-libs \
            libssh2 && \
    rm -rf /var/cache/apk/*

ARG MAJOR_VERSION=3.2
ARG ZBX_VERSION=${MAJOR_VERSION}.0
ARG ZBX_SOURCES=svn://svn.zabbix.com/tags/${ZBX_VERSION}/
ENV ZBX_VERSION=${ZBX_VERSION} ZBX_SOURCES=${ZBX_SOURCES} DB_TYPE=${DB_TYPE}

RUN apk add ${APK_FLAGS_DEV} --virtual build-dependencies \
            alpine-sdk \
            coreutils \
            automake \
            autoconf \
            sqlite-dev \
            libxml2-dev \
            unixodbc-dev \
            net-snmp-dev \
            libssh2-dev \
            openipmi-dev \
            openldap-dev \
            curl-dev \
            subversion && \
    cd /tmp/ && \
    svn --quiet export ${ZBX_SOURCES} zabbix-${ZBX_VERSION} 1>/dev/null && \
    cd /tmp/zabbix-${ZBX_VERSION} && \
    zabbix_revision=`svn info ${ZBX_SOURCES} |grep "Last Changed Rev"|awk '{print $4;}'` && \
    sed -i "s/{ZABBIX_REVISION}/$zabbix_revision/g" include/version.h && \
    ./bootstrap.sh 1>/dev/null && \
    ./configure \
            --prefix=/usr \
            --silent \
            --sysconfdir=/etc/zabbix \
            --libdir=/usr/lib/zabbix \
            --enable-proxy \
            --enable-ipv6 \
# Does not support stable iksemel library
#            --with-jabber \
            --with-ldap \
            --with-net-snmp \
            --with-openipmi \
            --with-ssh2 \
            --with-libcurl \
            --with-unixodbc \
            --with-libxml2 \
            --with-openssl \
            --with-${DB_TYPE} && \
    make -j"$(nproc)" -s dbschema 1>/dev/null && \
    make -j"$(nproc)" -s 1>/dev/null && \
    cp src/zabbix_proxy/zabbix_proxy /usr/sbin/zabbix_proxy && \
    cp conf/zabbix_proxy.conf /etc/zabbix/zabbix_proxy.conf && \
    sed -i -e "/^FpingLocation=/s/=.*/=\/usr\/sbin\/fping/" "/etc/zabbix/zabbix_proxy.conf" && \
    sed -i -e "/^Fping6Location=/s/=.*/=\/usr\/sbin\/fping6/" "/etc/zabbix/zabbix_proxy.conf" && \
    chown --quiet -R zabbix:root /etc/zabbix && \
    cd /tmp/ && \
    rm -rf /tmp/zabbix-${ZBX_VERSION}/ && \
    apk del ${APK_FLAGS_COMMON} --purge \
            build-dependencies && \
    rm -rf /var/cache/apk/*

EXPOSE 10051/TCP 162/UDP

WORKDIR /var/lib/zabbix

VOLUME ["/usr/lib/zabbix/externalscripts", "/var/lib/zabbix/enc", "/var/lib/zabbix/modules", "/var/lib/zabbix/ssh_keys"]
VOLUME ["/var/lib/zabbix/ssl/certs", "/var/lib/zabbix/ssl/keys", "/var/lib/zabbix/ssl/ssl_ca", "/var/lib/zabbix/snmptraps", "/var/lib/zabbix/mibs"]

ADD conf/etc/supervisor/ /etc/supervisor/
ADD run_zabbix_component.sh /

ENTRYPOINT ["/bin/bash"]

CMD ["/run_zabbix_component.sh", "proxy", "sqlite3"]
