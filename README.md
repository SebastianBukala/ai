cat > docker-compose.yml << 'EOF'
version: '3.9'
services:
  images_db:
    image: mariadb:10.11
    container_name: images_db
    restart: unless-stopped
    volumes:
      - /apps/zabbix/mysql-data:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: zabbix
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix
    ports:
      - "3306:3306"
    command: --max-connections=1000 --innodb-buffer-pool-size=256M

  images_zabbix_server:
    image: zabbix/zabbix-server-mysql:alpine-latest
    container_name: images_zabbix_server
    restart: unless-stopped
    volumes:
      - /usr/lib/zabbix/externalscripts:/usr/lib/zabbix/externalscripts:ro
    environment:
      DB_SERVER_HOST: images_db
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix
      ZBX_JAVAGATEWAY: images_zabbix_java
      ZBX_STARTPOLLERS: 100
      ZBX_CACHESIZE: 512M
      ZBX_HISTORYCACHESIZE: 128M
    depends_on:
      - images_db
    ports:
      - "10051:10051"

  images_web:
    image: zabbix/zabbix-web-nginx-mysql:alpine-latest
    container_name: images_web
    restart: unless-stopped
    environment:
      ZBX_SERVER_HOST: images_zabbix_server
      DB_SERVER_HOST: images_db
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix
      PHP_TZ: Europe/Warsaw
    ports:
      - "8080:8080"
    depends_on:
      - images_zabbix_server

  images_zabbix_java:
    image: zabbix/zabbix-java-gateway:alpine-latest
    container_name: images_zabbix_java
    restart: unless-stopped
    ports:
      - "10052:10052"
EOF
