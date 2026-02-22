# Домашнее задание: RPM в своем репозитории
## Задание  
1) Создать свой RPM пакет (можно взять свое приложение, либо собрать, например,
Apache с определенными опциями).  
2) Создать свой репозиторий и разместить там ранее собранный RPM.  
  
## Выполнение:  
# Задание 1.  
Обновим систему и установим необходимые пакеты:  
```
apt update
apt install -y \
  build-essential \
  git \
  cmake \
  wget \
  curl \
  ca-certificates \
  libpcre3 libpcre3-dev \
  zlib1g zlib1g-dev \
  libssl-dev
apt install devscripts
```

Обновим репозиторий, добавив источники deb-src, а также загрузим исходные файлы nginx и необходимые зависимости для сборки пакета:  
```
sed -i 's/^Types: deb$/Types: deb deb-src/' /etc/apt/sources.list.d/ubuntu.sources
apt update

mkdir deb && cd deb
apt source nginx

ls -al
total 1188
drwxr-xr-x  3 root root    4096 Feb 22 20:20 .
drwx------  6 root root    4096 Feb 22 20:19 ..
drwxr-xr-x 10 root root    4096 Feb 22 20:19 nginx-1.24.0
-rw-r--r--  1 root root   85120 Feb 12 22:51 nginx_1.24.0-2ubuntu7.6.debian.tar.xz
-rw-r--r--  1 root root    3617 Feb 12 22:51 nginx_1.24.0-2ubuntu7.6.dsc
-rw-r--r--  1 root root 1112471 Jun 28  2023 nginx_1.24.0.orig.tar.gz

apt build-dep nginx
```

Склонируем исходные файлы модуля и соберем его:  
```
git clone --recurse-submodules -j8 \
https://github.com/google/ngx_brotli

cd ngx_brotli/deps/brotli
mkdir out && cd out

cmake -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=OFF -DCMAKE_C_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_CXX_FLAGS="-Ofast -m64 -march=native -mtune=native -flto -funroll-loops -ffunction-sections -fdata-sections -Wl,--gc-sections" -DCMAKE_INSTALL_PREFIX=./installed ..

cmake --build . --config Release -j 2 --target brotlienc
```

Перейдем в директорию с исходными файлами nginx и поправим файл rules, в котором описаны дополнительные ключи для сборки:  
```
nano nginx-1.24.0/debian/rules
...
basic_configure_flags := \
                        --prefix=/usr/share/nginx \
                        --conf-path=/etc/nginx/nginx.conf \
                        --http-log-path=/var/log/nginx/access.log \
                        --error-log-path=stderr \
                        --lock-path=/var/lock/nginx.lock \
                        --pid-path=/run/nginx.pid \
                        --modules-path=/usr/lib/nginx/modules \
                        --http-client-body-temp-path=/var/lib/nginx/body \
                        --http-fastcgi-temp-path=/var/lib/nginx/fastcgi \
                        --http-proxy-temp-path=/var/lib/nginx/proxy \
                        --http-scgi-temp-path=/var/lib/nginx/scgi \
                        --http-uwsgi-temp-path=/var/lib/nginx/uwsgi \
                        --with-compat \
                        --add-module=/root/deb/ngx_brotli \
...
```  
Оставим комментарий в changelog версии и соберем новую версию nginx:  

```
dch -i

debuild -us -uc -b
 dpkg-buildpackage -us -uc -ui -b
dpkg-buildpackage: info: source package nginx
dpkg-buildpackage: info: source version 1.24.0-2ubuntu7.7
dpkg-buildpackage: info: source distribution UNRELEASED
dpkg-buildpackage: info: source changed by root <root@efilimonova-infra.rvision.local>
 dpkg-source --before-build .
dpkg-buildpackage: info: host architecture amd64
 debian/rules clean
dh clean --without autoreconf
   debian/rules override_dh_clean
make[1]: Entering directory '/root/deb/nginx-1.24.0'
...

ls -al
total 5164
drwxr-xr-x  4 root root    4096 Feb 22 20:32 .
drwx------  5 root root    4096 Feb 22 20:29 ..
-rw-r--r--  1 root root   22336 Feb 22 20:32 libnginx-mod-http-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root   36264 Feb 22 20:32 libnginx-mod-http-geoip-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root   26398 Feb 22 20:32 libnginx-mod-http-image-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root   43642 Feb 22 20:32 libnginx-mod-http-image-filter-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root   34970 Feb 22 20:32 libnginx-mod-http-perl_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root  107838 Feb 22 20:32 libnginx-mod-http-perl-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root   24780 Feb 22 20:32 libnginx-mod-http-xslt-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root   53376 Feb 22 20:32 libnginx-mod-http-xslt-filter-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root   57712 Feb 22 20:32 libnginx-mod-mail_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root  106192 Feb 22 20:32 libnginx-mod-mail-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root   84860 Feb 22 20:32 libnginx-mod-stream_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root  174518 Feb 22 20:32 libnginx-mod-stream-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root   21470 Feb 22 20:32 libnginx-mod-stream-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root   22312 Feb 22 20:32 libnginx-mod-stream-geoip-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
drwxr-xr-x 10 root root    4096 Feb 22 20:19 nginx-1.24.0
-rw-r--r--  1 root root   85120 Feb 12 22:51 nginx_1.24.0-2ubuntu7.6.debian.tar.xz
-rw-r--r--  1 root root    3617 Feb 12 22:51 nginx_1.24.0-2ubuntu7.6.dsc
-rw-r--r--  1 root root  371057 Feb 22 20:32 nginx_1.24.0-2ubuntu7.7_amd64.build
-rw-r--r--  1 root root   16486 Feb 22 20:32 nginx_1.24.0-2ubuntu7.7_amd64.buildinfo
-rw-r--r--  1 root root    9456 Feb 22 20:32 nginx_1.24.0-2ubuntu7.7_amd64.changes
-rw-r--r--  1 root root  963526 Feb 22 20:32 nginx_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root 1112471 Jun 28  2023 nginx_1.24.0.orig.tar.gz
-rw-r--r--  1 root root   43582 Feb 22 20:32 nginx-common_1.24.0-2ubuntu7.7_all.deb
-rw-r--r--  1 root root   17018 Feb 22 20:32 nginx-core_1.24.0-2ubuntu7.7_all.deb
-rw-r--r--  1 root root 1566270 Feb 22 20:32 nginx-dbgsym_1.24.0-2ubuntu7.7_amd64.ddeb
-rw-r--r--  1 root root  122106 Feb 22 20:32 nginx-dev_1.24.0-2ubuntu7.7_all.deb
-rw-r--r--  1 root root   25004 Feb 22 20:32 nginx-doc_1.24.0-2ubuntu7.7_all.deb
-rw-r--r--  1 root root   17258 Feb 22 20:32 nginx-extras_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r--  1 root root   17082 Feb 22 20:32 nginx-full_1.24.0-2ubuntu7.7_all.deb
-rw-r--r--  1 root root   16796 Feb 22 20:32 nginx-light_1.24.0-2ubuntu7.7_all.deb
drwxr-xr-x  7 root root    4096 Feb 22 20:23 ngx_brotli

```

Установим nginx, поправим конфиг и уберем оттуда IPv6 поддержку. Проверим статус службы:  
```
apt install /root/deb/nginx-full_1.24.0-2ubuntu7.7_all.deb

nano /etc/nginx/sites-enabled/default
systemctl restart nginx

systemctl status nginx
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since Sun 2026-02-22 20:44:22 MSK; 5s ago
       Docs: man:nginx(8)
    Process: 15988 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
    Process: 15990 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
   Main PID: 15991 (nginx)
      Tasks: 3 (limit: 4609)
     Memory: 2.4M (peak: 2.7M)
        CPU: 17ms
     CGroup: /system.slice/nginx.service
             ├─15991 "nginx: master process /usr/sbin/nginx -g daemon on; master_process on;"
             ├─15992 "nginx: worker process"
             └─15993 "nginx: worker process"

nginx -V
nginx version: nginx/1.24.0 (Ubuntu)
built with OpenSSL 3.0.13 30 Jan 2024
TLS SNI support enabled
configure arguments: --with-cc-opt='-g -O2 -fno-omit-frame-pointer -mno-omit-leaf-frame-pointer -ffile-prefix-map=/root/deb/nginx-1.24.0=. -flto=auto -ffat-lto-objects -fstack-protector-strong -fstack-clash-protection -Wformat -Werror=format-security -fcf-protection -fdebug-prefix-map=/root/deb/nginx-1.24.0=/usr/src/nginx-1.24.0-2ubuntu7.7 -fPIC -Wdate-time -D_FORTIFY_SOURCE=3' --with-ld-opt='-Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -Wl,-z,relro -Wl,-z,now -fPIC' --prefix=/usr/share/nginx --conf-path=/etc/nginx/nginx.conf --http-log-path=/var/log/nginx/access.log --error-log-path=stderr --lock-path=/var/lock/nginx.lock --pid-path=/run/nginx.pid --modules-path=/usr/lib/nginx/modules --http-client-body-temp-path=/var/lib/nginx/body --http-fastcgi-temp-path=/var/lib/nginx/fastcgi --http-proxy-temp-path=/var/lib/nginx/proxy --http-scgi-temp-path=/var/lib/nginx/scgi --http-uwsgi-temp-path=/var/lib/nginx/uwsgi --with-compat --add-module=/root/deb/ngx_brotli --with-debug --with-pcre-jit --with-http_ssl_module --with-http_stub_status_module --with-http_realip_module --with-http_auth_request_module --with-http_v2_module --with-http_dav_module --with-http_slice_module --with-threads --with-http_addition_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_secure_link_module --with-http_sub_module --with-mail_ssl_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-stream_realip_module --with-http_geoip_module=dynamic --with-http_image_filter_module=dynamic --with-http_perl_module=dynamic --with-http_xslt_module=dynamic --with-mail=dynamic --with-stream=dynamic --with-stream_geoip_module=dynamic
```

Создадим корневую директорию репозитория, скопируем туда deb файлы и соберем Package.gz со списком доступных пакетов:  
```
mkdir /usr/share/nginx/html/repo
cp *.deb /usr/share/nginx/html/repo/

ls -al /usr/share/nginx/html/repo/
total 1508
drwxr-xr-x 2 root root   4096 Feb 22 20:45 .
drwxr-xr-x 3 root root   4096 Feb 22 20:45 ..
-rw-r--r-- 1 root root  22336 Feb 22 20:45 libnginx-mod-http-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  26398 Feb 22 20:45 libnginx-mod-http-image-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  34970 Feb 22 20:45 libnginx-mod-http-perl_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  24780 Feb 22 20:45 libnginx-mod-http-xslt-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  57712 Feb 22 20:45 libnginx-mod-mail_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  84860 Feb 22 20:45 libnginx-mod-stream_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  21470 Feb 22 20:45 libnginx-mod-stream-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root 963526 Feb 22 20:45 nginx_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  43582 Feb 22 20:45 nginx-common_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  17018 Feb 22 20:45 nginx-core_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root 122106 Feb 22 20:45 nginx-dev_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  25004 Feb 22 20:45 nginx-doc_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  17258 Feb 22 20:45 nginx-extras_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  17082 Feb 22 20:45 nginx-full_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  16796 Feb 22 20:45 nginx-light_1.24.0-2ubuntu7.7_all.deb

dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
ls -al
total 1516
drwxr-xr-x 2 root root   4096 Feb 22 20:50 .
drwxr-xr-x 3 root root   4096 Feb 22 20:45 ..
-rw-r--r-- 1 root root  22336 Feb 22 20:45 libnginx-mod-http-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  26398 Feb 22 20:45 libnginx-mod-http-image-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  34970 Feb 22 20:45 libnginx-mod-http-perl_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  24780 Feb 22 20:45 libnginx-mod-http-xslt-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  57712 Feb 22 20:45 libnginx-mod-mail_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  84860 Feb 22 20:45 libnginx-mod-stream_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  21470 Feb 22 20:45 libnginx-mod-stream-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root 963526 Feb 22 20:45 nginx_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  43582 Feb 22 20:45 nginx-common_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  17018 Feb 22 20:45 nginx-core_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root 122106 Feb 22 20:45 nginx-dev_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  25004 Feb 22 20:45 nginx-doc_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  17258 Feb 22 20:45 nginx-extras_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  17082 Feb 22 20:45 nginx-full_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  16796 Feb 22 20:45 nginx-light_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root   4588 Feb 22 20:50 Packages.gz
```

Изменим конфиг nginx - поменяем корневую папку и добавим autoindex on:    
```
nano /etc/nginx/nginx.conf
nginx -t
nginx -s reload

 curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="Packages.gz">Packages.gz</a>                                        22-Feb-2026 17:50                4588
<a href="libnginx-mod-http-geoip_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-http-geoip_1.24.0-2ubuntu7.7_amd64..&gt;</a> 22-Feb-2026 17:45               22336
<a href="libnginx-mod-http-image-filter_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-http-image-filter_1.24.0-2ubuntu7...&gt;</a> 22-Feb-2026 17:45               26398
<a href="libnginx-mod-http-perl_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-http-perl_1.24.0-2ubuntu7.7_amd64.deb</a> 22-Feb-2026 17:45               34970
<a href="libnginx-mod-http-xslt-filter_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-http-xslt-filter_1.24.0-2ubuntu7.7..&gt;</a> 22-Feb-2026 17:45               24780
<a href="libnginx-mod-mail_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-mail_1.24.0-2ubuntu7.7_amd64.deb</a>      22-Feb-2026 17:45               57712
<a href="libnginx-mod-stream-geoip_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-stream-geoip_1.24.0-2ubuntu7.7_amd..&gt;</a> 22-Feb-2026 17:45               21470
<a href="libnginx-mod-stream_1.24.0-2ubuntu7.7_amd64.deb">libnginx-mod-stream_1.24.0-2ubuntu7.7_amd64.deb</a>    22-Feb-2026 17:45               84860
<a href="nginx-common_1.24.0-2ubuntu7.7_all.deb">nginx-common_1.24.0-2ubuntu7.7_all.deb</a>             22-Feb-2026 17:45               43582
<a href="nginx-core_1.24.0-2ubuntu7.7_all.deb">nginx-core_1.24.0-2ubuntu7.7_all.deb</a>               22-Feb-2026 17:45               17018
<a href="nginx-dev_1.24.0-2ubuntu7.7_all.deb">nginx-dev_1.24.0-2ubuntu7.7_all.deb</a>                22-Feb-2026 17:45              122106
<a href="nginx-doc_1.24.0-2ubuntu7.7_all.deb">nginx-doc_1.24.0-2ubuntu7.7_all.deb</a>                22-Feb-2026 17:45               25004
<a href="nginx-extras_1.24.0-2ubuntu7.7_amd64.deb">nginx-extras_1.24.0-2ubuntu7.7_amd64.deb</a>           22-Feb-2026 17:45               17258
<a href="nginx-full_1.24.0-2ubuntu7.7_all.deb">nginx-full_1.24.0-2ubuntu7.7_all.deb</a>               22-Feb-2026 17:45               17082
<a href="nginx-light_1.24.0-2ubuntu7.7_all.deb">nginx-light_1.24.0-2ubuntu7.7_all.deb</a>              22-Feb-2026 17:45               16796
<a href="nginx_1.24.0-2ubuntu7.7_amd64.deb">nginx_1.24.0-2ubuntu7.7_amd64.deb</a>                  22-Feb-2026 17:45              963526
</pre><hr></body>
</html>
```

Добавим наш новый репозиторий, проверим доступность пакетов:  
```
tee /etc/apt/sources.list.d/custom-nginx.list <<EOF
deb [trusted=yes] http://10.99.217.6/repo ./
EOF

apt update
apt-cache policy nginx
nginx:
  Installed: 1.24.0-2ubuntu7.7
  Candidate: 1.24.0-2ubuntu7.7
  Version table:
 *** 1.24.0-2ubuntu7.7 500
        500 http://10.99.217.6/repo ./ Packages
        100 /var/lib/dpkg/status
     1.24.0-2ubuntu7.6 500
        500 http://ru.archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
        500 http://security.ubuntu.com/ubuntu noble-security/main amd64 Packages
     1.24.0-2ubuntu7 500
        500 http://ru.archive.ubuntu.com/ubuntu noble/main amd64 Packages

```

Загрузим новый пакет, пересоздадим Packages.gz и проверим, что мы можем установить пакет:  
```
wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb

ls -al
total 1536
drwxr-xr-x 2 root root   4096 Feb 22 21:10 .
drwxr-xr-x 3 root root   4096 Feb 22 20:45 ..
-rw-r--r-- 1 root root  22336 Feb 22 20:45 libnginx-mod-http-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  26398 Feb 22 20:45 libnginx-mod-http-image-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  34970 Feb 22 20:45 libnginx-mod-http-perl_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  24780 Feb 22 20:45 libnginx-mod-http-xslt-filter_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  57712 Feb 22 20:45 libnginx-mod-mail_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  84860 Feb 22 20:45 libnginx-mod-stream_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  21470 Feb 22 20:45 libnginx-mod-stream-geoip_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root 963526 Feb 22 20:45 nginx_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  43582 Feb 22 20:45 nginx-common_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  17018 Feb 22 20:45 nginx-core_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root 122106 Feb 22 20:45 nginx-dev_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  25004 Feb 22 20:45 nginx-doc_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  17258 Feb 22 20:45 nginx-extras_1.24.0-2ubuntu7.7_amd64.deb
-rw-r--r-- 1 root root  17082 Feb 22 20:45 nginx-full_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root  16796 Feb 22 20:45 nginx-light_1.24.0-2ubuntu7.7_all.deb
-rw-r--r-- 1 root root   4588 Feb 22 20:50 Packages.gz
-rw-r--r-- 1 root root  18302 Aug 21  2025 percona-release_latest.noble_all.deb

dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz
apt update
apt-cache policy percona-release
percona-release:
  Installed: (none)
  Candidate: 1.0-32.generic
  Version table:
     1.0-32.generic 500
        500 http://10.99.217.6/repo ./ Packages

apt install percona-release -y
```