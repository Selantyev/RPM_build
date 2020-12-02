# RPM_build
(CentOS 8)

**Создать свой RPM пакет**
1. Устанавливаем пакеты

`# yum install -y redhat-lsb-core wget rpmdevtools rpm-build createrepo yum-utils tree`

2. Скачаем srpm

`$ wget https://cbs.centos.org/kojifiles/packages/haproxy/2.2.2/1.el8/src/haproxy-2.2.2-1.el8.src.rpm`

3. Устанавливаем пакет

`$ rpm -i haproxy-2.2.2-1.el8.src.rpm`

4. Проверяем, что в домашней директории создался каталог для сборки

```
$ tree rpmbuild/
rpmbuild/
├── SOURCES
│   ├── halog.1
│   ├── haproxy-2.2.2.tar.gz
│   ├── haproxy.cfg
│   ├── haproxy.logrotate
│   ├── haproxy.service
│   └── haproxy.sysconfig
└── SPECS
    └── haproxy.spec

2 directories, 7 file
```

5. Установим зависимости

```
# yum-builddep rpmbuild/SPECS/haproxy.spec`
# yum install -y systemd-devel pcre2-devel openssl-devel
# rpm -i http://mirror.centos.org/centos/8/PowerTools/x86_64/os/Packages/lua-devel-5.3.4-11.el8.x86_64.rpm
```

6. Добавим экстра опции в haproxy.spec

`CPU="native" USE_LINUX_SPLICE=1`

7. Запустим сборку

`$ rpmbuild -bb rpmbuild/SPECS/haproxy.spec`

8. Убедимся, что пакеты собраны

```
$ tree rpmbuild/RPMS/
rpmbuild/RPMS/
└── x86_64
    ├── haproxy-2.2.2-1.el8.x86_64.rpm
    ├── haproxy-debuginfo-2.2.2-1.el8.x86_64.rpm
    └── haproxy-debugsource-2.2.2-1.el8.x86_64.rpm

1 directory, 3 files
```

9. Установим собранный пакет

`# yum localinstall -y ~/rpmbuild/RPMS/x86_64/haproxy-2.2.2-1.el8.x86_64.rpm`

10. Запустим сервис haproxy.service

`# systemctl start haproxy.service`

```
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2020-12-02 12:53:05 UTC; 2min 20s ago
```
   
**Создать свой репозиторий и разместить там ранее собранный RPM**
1. Установим nginx

`# yum install -y nginx`

2. Создадим каталог repo в /usr/share/nginx/html

`# mkdir /usr/share/nginx/html/repo`

3. Копируем пакет в каталог /usr/share/nginx/html/repo

`cp rpmbuild/RPMS/x86_64/haproxy-2.2.2-1.el8.x86_64.rpm /usr/share/nginx/html/repo`

4. Скачаем в каталог /usr/share/nginx/html/repo rabbitmq

`# wget https://bintray.com/rabbitmq/rpm/download_file?file_path=rabbitmq-server%2Fv3.8.x%2Fel%2F6%2Fnoarch%2Frabbitmq-server-3.8.9-1.el6.noarch.rpm -O /usr/share/nginx/html/repo/l6.noarch.rpm -O /usr/share/nginx/html/repo/rabbitmq-server-3.8.9-1.el6.noarch.rpm`

5. Инициализируем репозиторий

`# createrepo /usr/share/nginx/html/repo/`

6. Настроим в NGINX доступ к листингу каталога. !!! Права 755 для каталогов, 644 для файлов !!!

`# vi /etc/nginx/conf.d/default.conf`

```
server {
        listen   80;
        server_name  localhost;
        access_log  /var/log/nginx/access.log;
        root   /usr/share/nginx/html/;
        location / {
                index  index.php index.html index.htm;
        }
        location /repo {
               autoindex on;
        }
		location /repo/repodata {
               autoindex on;
        }
}
```

7. Проверим конфигурацию

`# nginx -t`

8. Запустим сервис nginx.service

`systemctl start nginx.service`

9. Проверим через браузер

`# curl -a http://localhost/repo/`

```
<html>
<head><title>Index of /repo/</title></head>
<body bgcolor="white">
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          02-Dec-2020 17:27                   -
<a href="haproxy-2.2.2-1.el8.x86_64.rpm">haproxy-2.2.2-1.el8.x86_64.rpm</a>                     02-Dec-2020 17:16             1959656
<a href="rabbitmq-server-3.8.9-1.el6.noarch.rpm">rabbitmq-server-3.8.9-1.el6.noarch.rpm</a>             24-Sep-2020 18:47            15525517
</pre><hr></body>
</html>
```

10. Добавим репозиторий

`vi /etc/yum.repos.d/repo.repo`

```
[repo]
name=repo
baseurl=http://localhost/repo
gpgcheck=0
enabled=1
```

11. Обновим кэш

`# yum makecahe`

12. Установим rabbitmq (haproxy уже установлен)

`# yum install rabbitmq-server-3.8.9-1.el6.noarch`
