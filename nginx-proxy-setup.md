# Прокси-сервер Nginx с авторизацией Basic Auth

Пошаговое руководство по развертыванию.

## Шаг 1. Создание директории и файлов конфигурации

Создайте новую директорию для проекта и добавьте следующие файлы:

- `docker-compose.yml`

- `nginx.conf`

## Шаг 2. Подготовка паролей Basic Auth.

В созданной директории с конфигурацией **nginx** должен лежать файл с
хешами паролей `.htpasswd` 
Данный файл можно заранее подготовить на другой машине и перенести на целевую.

**Ниже описаны шаги для создания файла:**

1\. Установите пакет `apache2-utils` (если еще не установлен),
необходимый для создания файлов паролей `.htpasswd`:

- sudo apt update && sudo apt install -y apache2-utils

2\. В директории с конфигурацией **nginx** выполните команду для
создания файла `.htpasswd` и добавления пользователя:

- htpasswd -c .htpasswd test

> **Примечание:** замените `test` на нужное имя пользователя. После
> выполнения команды система запросит ввод и подтверждение пароля в
> интерактивном режиме.

При необходимости добавления второго пользователя использовать htpasswd
.htpasswd test2 (без -c)

## Шаг 3. Конфигурация `docker-compose.yml`

Отредактируйте файл `docker-compose.yml`, добавив путь к файлу
`.htpasswd` в секцию `volumes`.

**Итоговый файл** `docker-compose.yml`:

    version: "3.3"

    services:
      nginx-integra-s:
        image: docker-nexus.integra-s.com/nginx:1.0.2
        restart: always
        network_mode: host # при этом режиме порты слушаются напрямую в хосте, поэтому секция “ports:” не обязательна. 
        volumes:
          - ./cert:/etc/nginx/certs/ # Папка с сертификатами
          - ./nginx.conf:/etc/nginx/nginx.conf # Конфигурационный файл nginx.conf
          - ./integra-vol:/external_folder # Папка для логов
          - ./.htpasswd:/etc/nginx/.htpasswd # Файл авторизации 

## Шаг 4. Конфигурация `nginx.conf`

В главном конфигурационном файле `nginx.conf` необходимо описать
правила проксирования порта.

В нашем примере обращения на порт **3001** будут перенаправляться на
http://192.168.10.197:**3003**. В качестве примера конкретного
эндпоинта, до которого доходит запрос через прокси, используется
**/ffpost** --- то есть запрос вида http://\<nginx\>:3001/ffpost уходит
на http://192.168.10.197:3003/ffpost.

Для этого необходимо отредактировать файл конфигурации, заполнив секцию
`http{ ... } и добавив секцию server:`

**Итоговый файл** `nginx.conf`**будет выглядеть так:**

    worker_processes auto;

    events {
        worker_connections 1024;
    }

    http {
        include mime.types;
        include ./sites-enabled/*;
        default_type application/octet-stream;

        sendfile on;
        keepalive_timeout 65;

        access_log /external_folder/logs/access.log;
        error_log /external_folder/logs/error.log emerg;

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Real-Host $http_host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        server_tokens off;

        server {
            listen 3001; #Принимаем запросы на порт 3001 nginx
            server_name _;

            # авторизация Basic Auth
            auth_basic "Restricted area AMJS";
            auth_basic_user_file /etc/nginx/.htpasswd;

            location / {
                          #Пересылаем запросы на amjs на порт 3003
                proxy_pass http://192.168.10.197:3003;

                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

          # Поддержка больших POST-запросов (если передаются файлы, например FF POST)
                client_max_body_size 100M;
            }
        }
    }

## Шаг 5. Если необходим вариант с ssl

Создать в директории проекта папку **cert** и переместить в нее нужные
сертификаты.

**Добавить в файл nginx.conf блок server**:

```nginx
server {
    listen 3002 ssl;
    server_name _;

    # Указываем путь до наших сертификатов
    ssl_certificate /etc/nginx/certs/my_server.ru.crt;
    ssl_certificate_key /etc/nginx/certs/my_server.ru.key;

    ssl_protocols TLSv1.3;

    # Basic Auth
    auth_basic "Restricted area AMJS";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://192.168.10.197:3003;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Поддержка больших POST-запросов (если передаются файлы, например FF POST)
        client_max_body_size 100M;
    }
}
```

## Шаг 6. Запуск контейнера

Запустите сборку и старт контейнера в фоновом режиме:

    sudo docker compose up -d

## Шаг 7. Проверка работы (пример запроса)

Команда для отправки запроса с авторизацией через `curl`:

`- curl \--request POST \--header \"Content-Type: application/json\"
  \--data \'{\"proc\": \"com.integra.ember.call_stored\", \"args\":
  \[\"graph.get\", \[\"db91c0cd-e756-4de4-b71c-bc1f67d9bd8c\", 0,
  1\]\]}\' http://test:test@192.168.10.192:3001/ffpost \| json_pp`

**где test:test** - ваш логин и пароль Basic Auth, созданный на шаге 3.

При выполнении запроса с ssl использовать протокол **https**
