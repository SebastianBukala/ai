version: '3.8'

services:
  mysql-server:
    image: mysql:8.0
    restart: always
    environment:
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbixpw  # Zmień!
      MYSQL_ROOT_PASSWORD: rootpw  # Zmień!
    volumes:
      - mysql_/var/lib/mysql
      - ./files:/docker-entrypoint-initdb.d  # init.sql z Zabbix tu

  zabbix-server:
    build:
      context: .
      dockerfile_inline: |  # Inline Dockerfile z Oracle ODBC
        FROM zabbix/zabbix-server-mysql:7.0-alpine
        
        LABEL maintainer="rwieczorek3@external.t-mobile.pl"
        
        ENV http_proxy="http://10.21.140.111:58888"
        ENV https_proxy="http://10.21.140.111:58888"
        
        USER root
        
        RUN apk --no-cache add libaio libnsl libtirpc libc6-compat curl unzip
        
        # Instant Client 21.13 Basic + ODBC
        RUN cd /tmp && \
            curl -SL -o basic.zip https://download.oracle.com/otn_software/linux/instantclient/211300/instantclient-basiclite-linux.x64-21.13.0.0.0dbru.zip && \
            unzip basic.zip && mv instantclient_21_13 /usr/lib/instantclient && rm basic.zip && \
            cd /usr/lib/instantclient && \
            ln -sf libclntsh.so.21.1 libclntsh.so && ln -sf libocci.so.21.1 libocci.so && \
            ln -sf libociicus.so libociicus.so && ln -sf libnnz21.so libnnz21.so && \
            ln -sf /usr/lib/libnsl.so.3 /usr/lib/libnsl.so.1 && \
            ln -sf /lib/libc.so.6 /usr/lib/libresolv.so.2 && \
            ln -sf /lib/ld-musl-x86_64.so.1 /usr/lib/ld-linux-x86-64.so.2 && \
            curl -SL -o odbc.zip https://download.oracle.com/otn_software/linux/instantclient/211300/instantclient-odbc-linux.x64-21.13.0.0.0dbru.zip && \
            unzip odbc.zip && mv instantclient_21_13/* /usr/lib/instantclient/ && rm -rf instantclient_21_13 odbc.zip
        
        ENV ORACLE_BASE=/usr/lib/instantclient
        ENV LD_LIBRARY_PATH=/usr/lib/instantclient
        ENV TNS_ADMIN=/usr/lib/instantclient
        ENV ORACLE_HOME=/usr/lib/instantclient
        
        USER 1997
        
    restart: always
    ports:
      - "10051:10051"
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbixpw
      ZBX_JAVAGATEWAY: zabbix-java-gateway  # Opcja
      http_proxy: "http://10.21.140.111:58888"
      https_proxy: "http://10.21.140.111:58888"
    depends_on:
      - mysql-server
    volumes:
      - zabbix_server_/var/lib/zabbix
      # Configs ODBC/Oracle z /apps/zabbix/files
      - ./files/odbcinst.ini:/etc/odbcinst.ini:ro
      - ./files/odbc.ini:/etc/odbc.ini:ro
      - ./files/tnsnames.ora:/usr/lib/instantclient/tnsnames.ora:ro
      # Opcja apk: - ./files/*.apk:/tmp/*.apk:ro

  zabbix-web:
    image: zabbix/zabbix-web-nginx-mysql:7.0-alpine
    restart: always
    ports:
      - "8080:8080"
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbixpw
      MYSQL_ROOT_PASSWORD: rootpw
      ZBX_SERVER_HOST: zabbix-server
    depends_on:
      - mysql-server
      - zabbix-server

volumes:
  mysql_
  zabbix_server_

