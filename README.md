# Домашнее задание к занятию «Кластеризация и балансировка нагрузки» - `Страхов Игорь`

---

### Задание 1

Запустил два simple python сервера на портах 8888 и 9999. Установил HAProxy, настроил балансировку Round-robin на 4 уровне.

1. Запустил python http серверы:
```bash
cd /opt/http1 && python3 -m http.server 8888 --bind 127.0.0.1 &
cd /opt/http2 && python3 -m http.server 9999 --bind 127.0.0.1 &
```

2. Конфигурационный файл HAProxy `/etc/haproxy/haproxy.cfg`:

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	mode	tcp
	option	tcplog
	option	dontlognull
	timeout connect 5000
	timeout client  50000
	timeout server  50000

listen stats
	bind		:888
	mode		http
	stats		enable
	stats uri	/stats
	stats refresh	5s
	stats realm	Haproxy\ Statistics

listen web_tcp
	bind :1325
	mode tcp
	balance roundrobin
	server s1 127.0.0.1:8888 check inter 3s
	server s2 127.0.0.1:9999 check inter 3s
```

3. Перезапустил HAProxy `sudo systemctl restart haproxy` и проверил curl-запросами — запросы чередуются между серверами:

![Скриншот-1](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task1-curl.png)

4. Страница статистики HAProxy на http://192.168.1.90:888/stats:

![Скриншот-2](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task1-stats.png)

---

### Задание 2

Запустил три simple python сервера на портах 8888, 9999 и 7777. Настроил балансировку Weighted Round Robin на 7 уровне: первый сервер — вес 2, второй — 3, третий — 4. HAProxy балансирует только трафик, адресованный домену example.local (ACL).

1. Запустил python http серверы:
```bash
cd /opt/http1 && python3 -m http.server 8888 --bind 127.0.0.1 &
cd /opt/http2 && python3 -m http.server 9999 --bind 127.0.0.1 &
cd /opt/http3 && python3 -m http.server 7777 --bind 127.0.0.1 &
```

2. Конфигурационный файл HAProxy `/etc/haproxy/haproxy.cfg`:

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
	timeout connect 5000
	timeout client  50000
	timeout server  50000

listen stats
	bind		:888
	mode		http
	stats		enable
	stats uri	/stats
	stats refresh	5s
	stats realm	Haproxy\ Statistics

frontend example
	mode http
	bind :8088
	acl ACL_example.local hdr(host) -i example.local
	use_backend web_servers if ACL_example.local

backend web_servers
	mode http
	balance roundrobin
	option httpchk
	http-check send meth GET uri /index.html
	server s1 127.0.0.1:8888 weight 2 check
	server s2 127.0.0.1:9999 weight 3 check
	server s3 127.0.0.1:7777 weight 4 check
```

3. Запросы с указанием домена example.local — балансировка работает, запросы распределяются по весам:

![Скриншот-3](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task2-with-domain.png)

4. Запросы без указания домена — HAProxy возвращает 503, так как ACL не срабатывает:

![Скриншот-4](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task2-without-domain.png)

---

### Задание 3*

Настроил связку HAProxy + Nginx. Nginx принимает запросы на порту 80: файлы .jpg отдаёт сам из /var/www/, остальные запросы проксирует на HAProxy (порт 8088), который балансирует между двумя python серверами.

1. Конфигурация Nginx `/etc/nginx/sites-available/example-local`:

```
server {
    listen 80;
    server_name example.local;

    access_log /var/log/nginx/example-access.log;
    error_log  /var/log/nginx/example-error.log;

    location ~* \.jpg$ {
        root /var/www;
    }

    location / {
        proxy_pass http://127.0.0.1:8088;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

2. Конфигурация HAProxy `/etc/haproxy/haproxy.cfg`:

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
	timeout connect 5000
	timeout client  50000
	timeout server  50000

listen stats
	bind		:888
	mode		http
	stats		enable
	stats uri	/stats
	stats refresh	5s
	stats realm	Haproxy\ Statistics

frontend example
	mode http
	bind :8088
	default_backend web_servers

backend web_servers
	mode http
	balance roundrobin
	option httpchk
	http-check send meth GET uri /index.html
	server s1 127.0.0.1:8888 check
	server s2 127.0.0.1:9999 check
```

3. Запрос jpg-файла — Nginx отдаёт его сам (Server: nginx):

![Скриншот-5](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task3-jpg.png)

4. Запрос html — проксируется через HAProxy на python серверы, ответы чередуются:

![Скриншот-6](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task3-proxy.png)

---

### Задание 4*

Запустил 4 simple python сервера: на портах 8881, 8882 — отдают страницу example1.local, на портах 8883, 8884 — example2.local. Настроил два бэкенда HAProxy и фронтенд с ACL-маршрутизацией по доменному имени.

1. Конфигурационный файл HAProxy `/etc/haproxy/haproxy.cfg`:

```
global
	log /dev/log	local0
	log /dev/log	local1 notice
	chroot /var/lib/haproxy
	stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
	stats timeout 30s
	user haproxy
	group haproxy
	daemon

defaults
	log	global
	mode	http
	option	httplog
	option	dontlognull
	timeout connect 5000
	timeout client  50000
	timeout server  50000

listen stats
	bind		:888
	mode		http
	stats		enable
	stats uri	/stats
	stats refresh	5s
	stats realm	Haproxy\ Statistics

frontend example
	mode http
	bind :8088
	acl ACL_example1 hdr(host) -i example1.local
	acl ACL_example2 hdr(host) -i example2.local
	use_backend web_servers_1 if ACL_example1
	use_backend web_servers_2 if ACL_example2

backend web_servers_1
	mode http
	balance roundrobin
	option httpchk
	http-check send meth GET uri /index.html
	server s1 127.0.0.1:8881 check
	server s2 127.0.0.1:8882 check

backend web_servers_2
	mode http
	balance roundrobin
	option httpchk
	http-check send meth GET uri /index.html
	server s3 127.0.0.1:8883 check
	server s4 127.0.0.1:8884 check
```

2. Запросы к example1.local — ответ приходит от бэкенда web_servers_1:

![Скриншот-7](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task4-example1.png)

3. Запросы к example2.local — ответ приходит от бэкенда web_servers_2:

![Скриншот-8](https://github.com/polovod1987/sys-pattern-homework/blob/main/img/task4-example2.png)
