# Задание 6. Настройка Rate Limiting

Количество запросов от одного из партнёров всё ещё снижает производительность приложения. Это влияет на опыт использования API других партнёров. Этой проблемой тоже нужно заняться.

Одно из возможных решений — динамическое масштабирование приложения, которое вы уже внедрили. Однако ресурсы системы не бесконечные. Поэтому необходимо предусмотреть дополнительную защиту на случай, если количество запросов от партнёра снова превысит оговорённое количество.

С этим поможет паттерн Rate Limiting.

## Что нужно сделать

1. Перед вами конфигурационный файл Nginx. Доработайте его таким образом, чтобы обеспечить ограничение количества отправляемых запросов. Должно быть не более 10 запросов в минуту.

```jsx
http {
   # Настройка upstream для балансировки нагрузки
   upstream backend_servers {
       server backend1.example.com;
       server backend2.example.com;
       server backend3.example.com;
   }

   server {
       listen 80;

       location / {
           proxy_pass http://backend_servers;
       }

   }
}
```

2. Сделайте так, чтобы при превышении лимита запросов клиент получал HTTP-ошибку с кодом `429`.

Сохраните обновлённый конфигурационный файл в директорию Task6 в рамках пул-реквеста.

## Настройка конфигурации NGINX

Согласно [ссылке](https://nginx.org/en/docs/http/ngx_http_limit_req_module.html?utm_source=chatgpt.com) нам следует указать:

```jsx
http {
    # Cоздаём shared-зону счётчиков
    # 10 запросов в минуту с одного клиента (по IP)
    limit_req_zone $binary_remote_addr zone=partner_limit:10m rate=10r/m;

    # Upstream – как было
    upstream backend_servers {
        server backend1.example.com;
        server backend2.example.com;
        server backend3.example.com;
    }

    server {
        listen 80;

        # Применяем ограничение
        limit_req zone=partner_limit burst=0 nodelay;
        limit_req_status 429;   # отдаём HTTP 429 Too Many Requests

        location / {
            proxy_pass http://backend_servers;
        }
    }
}
```

- `limit_req_zone` — хеш-таблица счётчиков (10 MB хватает ~160 k IP-адресов)
- `rate=10r/m` — не более 10 запросов в минуту
- `burst=0 nodelay` — запретить «взрыв» сразу нескольких запросов; лишние сразу отсекаются
- `limit_req_status 429` — Nginx вернёт 429 Too Many Requests при превышении
