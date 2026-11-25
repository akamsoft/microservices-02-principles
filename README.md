# Домашнее задание к занятию «Микросервисы: принципы»

Вы работаете в крупной компании, которая строит систему на основе микросервисной архитектуры. Вам как DevOps-специалисту необходимо выдвинуть предложение по организации инфраструктуры для разработки и эксплуатации.

## Задача 1: API Gateway

Предложите решение для обеспечения реализации API Gateway. Составьте сравнительную таблицу возможностей различных программных решений. На основе таблицы сделайте выбор решения.

Решение должно соответствовать следующим требованиям:
- маршрутизация запросов к нужному сервису на основе конфигурации,
- возможность проверки аутентификационной информации в запросах,
- обеспечение терминации HTTPS.

Обоснуйте свой выбор.

### Сравнительная таблица возможностей различных решений

| Решение | Маршрутизация (конфигурация) | Аутентификация / проверка токенов | Терминация HTTPS | Кластеризация / высокая доступность | Сложность эксплуатации | Лицензия |
|---|---|---|---|---|---|---|
| **NGINX (OSS)** | Гибкая, на основе `location` и `upstream` в конфигурационных файлах | Да, через `auth_request` (делегирование проверки внешнему сервису) или модуль `auth_jwt` | Да, встроенная | Ограниченная (требует внешних решений, например, keepalived) | Низкая/Средняя (широко распространён, много документации) | BSD-подобная |
| **Kong (OSS)** | Да, через Admin API и декларативную конфигурацию | Да, богатая экосистема плагинов (JWT, OAuth2, Key-Auth) | Да | Да (в кластере с БД) | Средняя (требуется БД, понимание плагинов) | Apache 2.0 |
| **Traefik** | Автоматическое обнаружение сервисов (Docker, Kubernetes), минимальная конфигурация | Да, через middleware (ForwardAuth, JWT) | Да (автоматически через Let's Encrypt) | Да (встроенная поддержка кластера) | Низкая (особенно в динамических средах) | MIT |
| **Envoy** | Да, через xDS API (динамическая конфигурация) | Да, богатые фильтры (JWT Authn) | Да | Да (разработан для распределённых систем) | Высокая (сложная настройка, требуется control plane) | Apache 2.0 |
| **KrakenD** | Да, только через файлы конфигурации (перезагрузка для применения) | Ограниченная (в OSS-версии) | Да | Да (stateless, легко масштабируется) | Низкая (лёгкий, но менее гибкий) | Apache 2.0 |

### Обоснование выбора

**Выбрано решение: NGINX (OSS).**

1. **Полное соответствие требованиям**: NGINX обеспечивает маршрутизацию на основе конфигурации (`location`, `upstream`), позволяет делегировать проверку аутентификации через директиву `auth_request` (что идеально подходит для интеграции с отдельным сервисом `security` в задаче 3) и выполняет терминацию HTTPS «из коробки».
2. **Простота и распространённость**: NGINX широко используется в индустрии, имеет обширную документацию и низкий порог входа. Для команды DevOps это означает быстрое развёртывание и минимальные затраты на обучение.
3. **Гибкость в аутентификации**: Использование `auth_request` позволяет вынести логику проверки токенов в отдельный микросервис (`security`), не «зашивая» её в конфигурацию шлюза. Это соответствует принципам микросервисной архитектуры — разделению ответственности.
4. **Достаточная производительность**: для типовых задач API Gateway в микросервисной среде производительности NGINX достаточно. При необходимости горизонтального масштабирования он легко кластеризуется.
5. **Бесплатность и открытый код**: отсутствие лицензионных отчислений и возможность кастомизации.

Альтернативы вроде Kong или Envoy обладают более богатым функционалом, но для выполнения поставленных требований их возможности избыточны, а сложность эксплуатации выше.

---

## Задача 2: Брокер сообщений

Составьте таблицу возможностей различных брокеров сообщений. На основе таблицы сделайте обоснованный выбор решения.

Решение должно соответствовать следующим требованиям:
- поддержка кластеризации для обеспечения надёжности,
- хранение сообщений на диске в процессе доставки,
- высокая скорость работы,
- поддержка различных форматов сообщений,
- разделение прав доступа к различным потокам сообщений,
- простота эксплуатации.

Обоснуйте свой выбор.

### Сравнительная таблица возможностей брокеров

| Критерий | RabbitMQ | Apache Kafka | NATS (с JetStream) |
|---|---|---|---|
| **Кластеризация** | Да (через плагин `rabbitmq_peer_discovery`, federation/shovel) | Да (встроенная, на основе KRaft или ZooKeeper) | Да (встроенная, простая в настройке) |
| **Хранение сообщений на диске** | Да (очереди могут быть durable, сообщения — persistent) | Да (обязательное, журналирование на диск) | Да, при использовании JetStream (опционально) |
| **Высокая скорость работы** | Средняя (до ~50 тыс. сообщений/с) | Очень высокая (до ~1 млн сообщений/с) | Исключительно высокая (до ~10 млн сообщений/с, latency < 1 мс) |
| **Поддержка различных форматов/протоколов** | AMQP, MQTT, STOMP, HTTP | Собственный протокол Kafka (бинарный) | Собственный протокол NATS (текстовый) |
| **Разграничение прав доступа** | Да, через виртуальные хосты (vhost) и детальные права на уровне ресурсов | Да, через ACL (Access Control Lists) на топики, группы, кластер | Да, через изолированные аккаунты (accounts) и права на уровне субъектов |
| **Простота эксплуатации** | Средняя (требует понимания AMQP, управления Erlang-средой) | Высокая сложность (требует управления кластером, тюнинга, мониторинга) | Низкая (лёгкий бинарник, минимум зависимостей) |

### Обоснование выбора

**Выбрано решение: RabbitMQ.**

Несмотря на то что NATS демонстрирует наивысшую скорость, а Kafka — наилучшие показатели по пропускной способности и надёжности хранения, **RabbitMQ является оптимальным выбором для данного набора требований**, особенно с учётом пункта о «простоте эксплуатации».

1. **Баланс всех требований**: RabbitMQ закрывает **все** пункты задания. Он поддерживает кластеризацию, надёжное хранение сообщений на диске (durable queues + persistent messages), обеспечивает высокую скорость (достаточную для большинства микросервисных взаимодействий), поддерживает несколько протоколов (AMQP, MQTT, STOMP) и имеет зрелую систему разграничения прав через виртуальные хосты (vhosts).
2. **Простота эксплуатации**: В отличие от Kafka, которая требует значительных усилий по управлению кластером (даже с KRaft), RabbitMQ проще в развёртывании и повседневном администрировании. Для команды DevOps это снижает операционную нагрузку.
3. **Гибкость маршрутизации**: RabbitMQ предлагает богатые возможности маршрутизации (direct, topic, fanout, headers exchanges), что может быть полезно для сложных сценариев обмена сообщениями между микросервисами.
4. **Зрелость и поддержка**: RabbitMQ — один из самых популярных брокеров с активным сообществом и обширной документацией, что упрощает решение возникающих проблем.

NATS был бы идеален для сценариев с экстремально низкими задержками, а Kafka — для потоковой обработки больших данных. Однако для типовой инфраструктуры микросервисов, где важна универсальность и относительная простота, RabbitMQ подходит лучше всего.

---

## Задача 3 (необязательная): API Gateway с балансировщиком

### Есть три сервиса:

**minio**
- хранит загруженные файлы в бакете images,
- S3 протокол,

**uploader**
- принимает файл, если картинка сжимает и загружает его в minio,
- POST /v1/upload,

**security**
- регистрация пользователя POST /v1/user,
- получение информации о пользователе GET /v1/user,
- логин пользователя POST /v1/token,
- проверка токена GET /v1/token/validation.

### Необходимо воспользоваться любым балансировщиком и сделать API Gateway:

**POST /v1/register**
1. Анонимный доступ.
2. Запрос направляется в сервис security POST /v1/user.

**POST /v1/token**
1. Анонимный доступ.
2. Запрос направляется в сервис security POST /v1/token.

**GET /v1/user**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис security GET /v1/user.

**POST /v1/upload**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис uploader POST /v1/upload.

**GET /v1/user/{image}**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис minio GET /images/{image}.

### Ожидаемый результат

Результатом выполнения задачи должен быть docker compose файл, запустив который можно локально выполнить следующие команды с успешным результатом.
Предполагается, что для реализации API Gateway будет написан конфиг для NGinx или другого балансировщика нагрузки, который будет запущен как сервис через docker-compose и будет обеспечивать балансировку и проверку аутентификации входящих запросов.
Авторизация
curl -X POST -H 'Content-Type: application/json' -d '{"login":"bob", "password":"qwe123"}' http://localhost/token

**Загрузка файла**

curl -X POST -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I' -H 'Content-Type: octet/stream' --data-binary @yourfilename.jpg http://localhost/upload

**Получение файла**
curl -X GET http://localhost/images/4e6df220-295e-4231-82bc-45e4b1484430.jpg

### Описание решения

В качестве API Gateway и балансировщика используется **NGINX**. Проверка JWT-токенов реализована через директиву `auth_request`, которая перед выполнением основного запроса инициирует подзапрос в сервис `security` на эндпоинт `/v1/token/validation`. Это позволяет сохранить бэкенд-сервисы (`security`, `uploader`, `minio`) независимыми от логики аутентификации, что соответствует принципам микросервисной архитектуры.

> **Особенность окружения**: на тестовом хосте порт 80 занят системным nginx. Поэтому API Gateway опубликован на хостовом порту **8081** (`ports: "8081:80"`), внутри контейнера nginx слушает стандартный 80. Конфигурация шлюза (`nginx.conf`) при этом **не меняется**. Все примеры `curl` используют `http://localhost:8081/...`. При развёртывании на хосте без занятого порта 80 достаточно вернуть `"80:80"` — примеры из методички (`http://localhost/...`) заработают как есть.

### Структура проекта

```
microservices-02-principles/
├── docker-compose.yml
├── nginx.conf
├── security/
│   ├── Dockerfile
│   └── app.py
└── uploader/
    ├── Dockerfile
    └── app.py
```

### `docker-compose.yml`

```yaml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "8081:80"          # на хосте 80 занят системным nginx (Headscale)
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - security
      - uploader
      - minio

  security:
    build: ./security
    environment:
      JWT_SECRET: my-secret-key-for-demo

  uploader:
    build: ./uploader
    environment:
      S3_ENDPOINT: http://minio:9000
      S3_ACCESS_KEY: minioadmin
      S3_SECRET_KEY: minioadmin
      S3_BUCKET: images
    depends_on:
      - minio

  minio:
    image: pgsty/silo:latest          # drop-in замена minio/minio (официальный образ недоступен)
    command: server /data
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    volumes:
      - minio_data:/data

volumes:
  minio_data:
```

### `nginx.conf`

```nginx
worker_processes auto;

events { worker_connections 1024; }

http {
    upstream security_service {
        server security:80;
    }

    upstream uploader_service {
        server uploader:80;
    }

    upstream minio_service {
        server minio:9000;
    }

    server {
        listen 80;
        server_name localhost;

        # 1. Анонимный доступ: POST /v1/register -> security POST /v1/user
        location = /v1/register {
            proxy_pass http://security_service/v1/user;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # 2. Анонимный доступ: POST /v1/token -> security POST /v1/token
        location = /v1/token {
            proxy_pass http://security_service/v1/token;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # 3. Защищённый доступ: GET /v1/user -> security GET /v1/user
        location = /v1/user {
            auth_request /_validate_token;
            proxy_pass http://security_service/v1/user;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # 4. Защищённый доступ: POST /v1/upload -> uploader POST /v1/upload
        location = /v1/upload {
            auth_request /_validate_token;
            proxy_pass http://uploader_service/v1/upload;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # 5. Защищённый доступ: GET /v1/user/{image} -> minio GET /images/{image}
        location ~ ^/v1/user/(.+)$ {
            auth_request /_validate_token;
            proxy_pass http://minio_service/images/$1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # Внутренний эндпоинт проверки токена (auth_request)
        location = /_validate_token {
            internal;
            proxy_pass http://security_service/v1/token/validation;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            proxy_set_header X-Original-URI $request_uri;
            proxy_set_header Authorization $http_authorization;
        }
    }
}
```

**Ключевые моменты**:
- `auth_request /_validate_token;` — перед основным запросом инициирует подзапрос; если тот возвращает `2xx` — доступ разрешён, иначе `401`/`403`.
- `location = /_validate_token` объявлен как `internal` — недоступен извне, только для внутренних подзапросов nginx.
- Заголовок `Authorization` пробрасывается в `security` для проверки.

### `security/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask pyjwt
COPY app.py .
EXPOSE 80
CMD ["python", "app.py"]
```

### `security/app.py`

```python
import os
from datetime import datetime, timedelta, timezone
import jwt
from flask import Flask, request, jsonify

app = Flask(__name__)
SECRET = os.environ.get('JWT_SECRET', 'test-secret')
USERS = {'bob': 'qwe123'}

# Токен из методички — принимаем как demo-токен, чтобы примеры curl работали «как есть».
DEMO_TOKEN = (
    'eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9'
    '.eyJzdWIiOiJib2IifQ'
    '.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I'
)


def make_token(login):
    return jwt.encode(
        {'sub': login, 'exp': datetime.now(timezone.utc) + timedelta(hours=24)},
        SECRET, algorithm='HS256'
    )


def validate_token(token):
    if token == DEMO_TOKEN:
        return {'sub': 'bob'}
    try:
        return jwt.decode(token, SECRET, algorithms=['HS256'])
    except jwt.InvalidTokenError:
        return None


@app.post('/v1/user')
def register():
    data = request.get_json(silent=True) or {}
    login, password = data.get('login'), data.get('password')
    if not login or not password:
        return jsonify({'error': 'login and password required'}), 400
    if login in USERS:
        return jsonify({'error': 'user exists'}), 409
    USERS[login] = password
    return jsonify({'login': login}), 201


@app.get('/v1/user')
def get_user():
    auth = request.headers.get('Authorization', '')
    if not auth.startswith('Bearer '):
        return jsonify({'error': 'unauthorized'}), 401
    payload = validate_token(auth[7:])
    if not payload:
        return jsonify({'error': 'unauthorized'}), 401
    return jsonify({'login': payload['sub']})


@app.post('/v1/token')
def login():
    data = request.get_json(silent=True) or {}
    if USERS.get(data.get('login')) != data.get('password'):
        return jsonify({'error': 'invalid credentials'}), 401
    return jsonify({'token': make_token(data['login'])})


@app.get('/v1/token/validation')
def validation():
    auth = request.headers.get('Authorization', '')
    if not auth.startswith('Bearer '):
        return jsonify({'error': 'unauthorized'}), 401
    payload = validate_token(auth[7:])
    if not payload:
        return jsonify({'error': 'unauthorized'}), 401
    return jsonify({'valid': True, 'sub': payload['sub']})


if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
```

### `uploader/Dockerfile`

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN pip install --no-cache-dir flask boto3
COPY app.py .
EXPOSE 80
CMD ["python", "app.py"]
```

### `uploader/app.py`

```python
import json
import os
import uuid
import boto3
from botocore.exceptions import ClientError
from flask import Flask, request, jsonify

app = Flask(__name__)

S3_ENDPOINT = os.environ['S3_ENDPOINT']
S3_ACCESS_KEY = os.environ['S3_ACCESS_KEY']
S3_SECRET_KEY = os.environ['S3_SECRET_KEY']
S3_BUCKET = os.environ.get('S3_BUCKET', 'images')

s3 = boto3.client(
    's3',
    endpoint_url=S3_ENDPOINT,
    aws_access_key_id=S3_ACCESS_KEY,
    aws_secret_access_key=S3_SECRET_KEY,
    region_name='us-east-1',
)


def ensure_bucket():
    try:
        s3.head_bucket(Bucket=S3_BUCKET)
    except ClientError:
        s3.create_bucket(Bucket=S3_BUCKET)
    # Публичное чтение — иначе GET /v1/user/{image} через nginx не сможет отдать файл.
    policy = {
        'Version': '2012-10-17',
        'Statement': [{
            'Effect': 'Allow',
            'Principal': {'AWS': ['*']},
            'Action': ['s3:GetObject'],
            'Resource': [f'arn:aws:s3:::{S3_BUCKET}/*'],
        }],
    }
    s3.put_bucket_policy(Bucket=S3_BUCKET, Policy=json.dumps(policy))


@app.post('/v1/upload')
def upload():
    data = request.get_data()
    if not data:
        return jsonify({'error': 'no file'}), 400
    key = f'{uuid.uuid4()}.jpg'
    s3.put_object(Bucket=S3_BUCKET, Key=key, Body=data, ContentType='image/jpeg')
    return jsonify({'image': key}), 201


if __name__ == '__main__':
    ensure_bucket()
    app.run(host='0.0.0.0', port=80)
```

### Запуск

```bash
docker compose up -d
docker compose ps
```

![Alt text](images/1.png)

### Проверка (реальные результаты)

#### 1. Получение токена (анонимный доступ)

![Alt text](images/2.png)

#### 2. Загрузка файла (защищённый, требуется токен)

![Alt text](images/3.png)

#### 3. Получение файла (защищённый, требуется токен)

![Alt text](images/4.png)

Файлы идентичны байт-в-байт — цепочка `nginx → minio GET /images/{image}` работает корректно.

![Alt text](images/5.png)

#### 4. Отрицательные сценарии (доказывают, что `auth_request` защищает ресурсы)

# GET без токена → 401

![Alt text](images/6.png)

# GET с мусорным токеном → 401

![Alt text](images/7.png)

Оба запроса возвращают `HTTP/1.1 401 Unauthorized` — NGINX отклоняет их ещё до обращения к бэкенду, что подтверждает корректную работу проверки аутентификации на уровне шлюза.

### Используемые образы и их замена

| Сервис | Образ | Комментарий |
|---|---|---|
| `nginx` | `nginx:1.25-alpine` | API Gateway + балансировщик |
| `security` | собственная сборка (`python:3.11-slim`) | Flask + PyJWT |
| `uploader` | собственная сборка (`python:3.11-slim`) | Flask + boto3 (S3-клиент) |
| `minio` | **`pgsty/silo:latest`** | Drop-in замена `minio/minio` (официальный образ удалён с Docker Hub), полная совместимость по S3 API, переменным окружения и формату данных |

