# HTTP

## Версии

| Версия | Год | Ключевые особенности |
|--------|-----|---------------------|
| HTTP/1.0 | 1996 | Новое соединение на каждый запрос |
| HTTP/1.1 | 1997 | Keep-alive, pipelining, chunked transfer |
| HTTP/2 | 2015 | Мультиплексирование, server push, сжатие заголовков (HPACK) |
| HTTP/3 | 2022 | QUIC (UDP), 0-RTT, нет head-of-line blocking |

## HTTP/1.1 vs HTTP/2

**HTTP/1.1 проблемы:**
- Head-of-line blocking — один медленный запрос блокирует остальные
- Текстовый протокол — overhead на парсинг
- Несжатые заголовки (повторяются в каждом запросе)

**HTTP/2 решения:**
- Бинарный фрейминг — запросы/ответы разбиваются на фреймы
- Мультиплексирование — множество потоков (streams) в одном TCP-соединении
- HPACK — сжатие заголовков (статическая + динамическая таблица)
- Server push — сервер отправляет ресурсы до запроса клиента

## Методы

| Метод | Идемпотентный | Безопасный | Тело | Описание |
|-------|:---:|:---:|:---:|----------|
| GET | Да | Да | Нет | Получить ресурс |
| HEAD | Да | Да | Нет | Только заголовки ответа |
| POST | Нет | Нет | Да | Создать ресурс |
| PUT | Да | Нет | Да | Заменить ресурс целиком |
| PATCH | Нет | Нет | Да | Частичное обновление |
| DELETE | Да | Нет | Нет | Удалить ресурс |
| OPTIONS | Да | Да | Нет | Доступные методы (CORS preflight) |

**Идемпотентность** — повторный вызов даёт тот же результат.
**Безопасность** — не изменяет состояние сервера.

## Коды ответа

| Диапазон | Категория | Примеры |
|----------|-----------|---------|
| 1xx | Информационные | 101 Switching Protocols |
| 2xx | Успех | 200 OK, 201 Created, 204 No Content |
| 3xx | Редиректы | 301 Moved, 302 Found, 304 Not Modified |
| 4xx | Ошибка клиента | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests |
| 5xx | Ошибка сервера | 500 Internal, 502 Bad Gateway, 503 Unavailable, 504 Gateway Timeout |

## Кеширование

```
Cache-Control: max-age=3600, public
Cache-Control: no-cache           # всегда валидируй с сервером
Cache-Control: no-store           # не кешируй вообще
ETag: "abc123"                    # fingerprint ресурса
If-None-Match: "abc123"           # условный запрос → 304 если не изменился
```

## Заголовки

```
Content-Type: application/json
Authorization: Bearer <token>
Accept: application/json
X-Request-ID: uuid              # для трейсинга
Retry-After: 60                 # при 429/503
```

## Keep-Alive и Connection Pooling

HTTP/1.1 по умолчанию keep-alive — TCP-соединение переиспользуется.

```go
// Go: настройка transport для connection pooling
transport := &http.Transport{
    MaxIdleConns:        100,
    MaxIdleConnsPerHost: 10,
    IdleConnTimeout:     90 * time.Second,
}
client := &http.Client{Transport: transport}
```

## CORS

Cross-Origin Resource Sharing — браузер блокирует запросы к другому origin.

```
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

Preflight запрос (OPTIONS) отправляется для «непростых» запросов.
