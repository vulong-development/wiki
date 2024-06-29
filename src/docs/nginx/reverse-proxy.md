---
 title: Настройка reverse proxy
 layout: base-layout
 time: 29 июня 2024 года
---

Вот пример конфигурации для nginx.conf, которую можно использовать в контейнере Docker для настройки Nginx в качестве обратного прокси для других контейнеров:

```nginx
http {
    server {
        listen 80;

        location /service1/ {
            proxy_pass http://service1_container:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /service2/ {
            proxy_pass http://service2_container:8080/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}

```

Здесь service1_container и service2_container - это имена контейнеров Docker для соответствующих сервисов, которые должны быть связаны с Nginx через Docker networking. Замените 8080 на порт, который слушает ваш сервис.

Чтобы использовать этот файл конфигурации в контейнере Docker, создайте Dockerfile:

```Docker
FROM nginx:latest

COPY nginx.conf /etc/nginx/nginx.conf

```

Создайте образ и запустите контейнер Nginx:

```Bash
docker build -t my-nginx .
docker run -d --name my-nginx-proxy --link service1_container --link service2_container -p 80:80 my-nginx
```

Здесь --link используется для устаревшего сетевого режима Docker. Если вы используете пользовательские сети Docker, используйте --net для подключения к той же сети, что и ваши сервисы, и убедитесь, что вы используете имена сервисов, определенные в вашей сети Docker.