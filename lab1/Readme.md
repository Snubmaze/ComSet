# Отчет по лабораторной работе №3: HA Postgres Cluster

## Часть 1. Развертывание PostgreSQL кластера

### Шаг 1. Создание Dockerfile

Создан `Dockerfile` для сборки образа с PostgreSQL и Patroni:
```dockerfile
FROM postgres:15

# Ставим нужные для Patroni зависимости
RUN apt-get update -y && \
 apt-get install -y netcat-openbsd python3-pip curl python3-psycopg2 python3-venv iputils-ping

# Используем виртуальное окружение, доустанавливаем, собственно, Patroni
RUN python3 -m venv /opt/patroni-venv && \
 /opt/patroni-venv/bin/pip install --upgrade pip && \
 /opt/patroni-venv/bin/pip install patroni[zookeeper] psycopg2-binary

# Копируем конфигурацию для двух узлов кластера Patroni
COPY postgres0.yml /postgres0.yml
COPY postgres1.yml /postgres1.yml

ENV PATH="/opt/patroni-venv/bin:$PATH"

USER postgres
```

### Шаг 2. docker-compose.yml

```yaml
services:
 pg-master:
   build: .
   image: localhost/postres:patroni
   container_name: pg-master
   restart: always
   hostname: pg-master
   environment:
     POSTGRES_USER: postgres
     POSTGRES_PASSWORD: postgres
     PGDATA: '/var/lib/postgresql/data/pgdata'
   expose:
     - 8008
   ports:
     - 5433:5432
   volumes:
     - pg-master:/var/lib/postgresql/data
   command: patroni /postgres0.yml

 pg-slave:
   build: .
   image: localhost/postres:patroni
   container_name: pg-slave
   restart: always
   hostname: pg-slave
   expose:
     - 8008
   ports:
     - 5434:5432
   volumes:
     - pg-slave:/var/lib/postgresql/data
   environment:
     POSTGRES_USER: postgres
     POSTGRES_PASSWORD: postgres
     PGDATA: '/var/lib/postgresql/data/pgdata'
   command: patroni /postgres1.yml

 zoo:
   image: confluentinc/cp-zookeeper:7.7.1
   container_name: zoo
   restart: always
   hostname: zoo
   ports:
     - 2181:2181
   environment:
     ZOOKEEPER_CLIENT_PORT: 2181
     ZOOKEEPER_TICK_TIME: 2000

volumes:
 pg-master:
 pg-slave:
```

### Шаг 3. Создание конфига Patroni

**Файл postgres0.yml:**
```yaml
scope: my_cluster
name: postgresql0
restapi:
 listen: pg-master:8008
 connect_address: pg-master:8008

zookeeper:
 hosts:
   - zoo:2181

bootstrap:
 dcs:
   ttl: 30
   loop_wait: 10
   retry_timeout: 10
   maximum_lag_on_failover: 10485760
   master_start_timeout: 300
   synchronous_mode: true
   postgresql:
     use_pg_rewind: true
     use_slots: true
     parameters:
       wal_level: replica
       hot_standby: "on"
       wal_keep_segments: 8
       max_wal_senders: 10
       max_replication_slots: 10
       wal_log_hints: "on"
       archive_mode: "always"
       archive_timeout: 1800s
       archive_command: mkdir -p /tmp/wal_archive && test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f
 pg_hba:
   - host replication replicator 0.0.0.0/0 md5
   - host all all 0.0.0.0/0 md5

postgresql:
 listen: 0.0.0.0:5432
 connect_address: pg-master:5432
 data_dir: /var/lib/postgresql/data/postgresql0
 bin_dir: /usr/lib/postgresql/15/bin
 pgpass: /tmp/pgpass0
 authentication:
   replication:
     username: replicator
     password: rep-pass
   superuser:
     username: postgres
     password: postgres
 parameters:
   unix_socket_directories: '.'

watchdog:
 mode: off

tags:
 nofailover: false
 noloadbalance: false
 clonefrom: false
 nosync: false
```

**Файл postgres1.yml** (отличия от postgres0.yml):
```yaml
scope: my_cluster
name: postgresql1
restapi:
 listen: pg-slave:8008
 connect_address: pg-slave:8008

# ... остальное идентично postgres0.yml ...

postgresql:
 listen: 0.0.0.0:5432
 connect_address: pg-slave:5432
 data_dir: /var/lib/postgresql/data/postgresql1
```

### Шаг 4. Запуск кластера

```bash
docker-compose up -d --build
```

Проверка логов:
```bash
docker-compose logs -f
```

**Результат:** Zookeeper успешно запустился, одна из нод PostgreSQL стала лидером.

![Логи запуска кластера - postgresql0 стал лидером](./lab1/images/leader.png)


А postgres1 - secondary

![Логи запуска кластера - postgresql1](./lab1/images/slave.png)

**Ответ на вопрос о пересборке образа:**
- При обычном `docker-compose up` образ НЕ будет пересобран
- Редактирование `postgresX.yml` НЕ приведет к пересборке (файлы копируются во время build)
- Изменение `Dockerfile` также НЕ приведет к автоматической пересборке
- Для пересборки необходимо использовать флаг `--build` или команду `docker-compose build`

---

## Часть 2. Проверка репликации

### Шаг 1. Подключение к нодам

Подключаемся к обеим нодам PostgreSQL через PG Admin:

На pg-master создаем таблицу users и заполняем данными:
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100),
);

INSERT INTO users (id, email) 
VALUES (1, 'test@gmail,.com'),
```

### Шаг 2. Проверка репликации на slave

На ноде pg-slave выполнен запрос:
```sql
SELECT * FROM users;
```

Видим, что данные успешно реплицировались на slave-ноду. Таблица и все записи появились автоматически.

### Шаг 3. Попытка записи на slave

![Ошибка при попытке записи на slave](./lab1/images/forbidden.png)

Получена ошибка, так как slave-нода работает в режиме read-only

---

## Часть 3. Настройка высокой доступности с HAProxy

### Шаг 1. Добавление HAProxy в docker-compose.yml

В файл `docker-compose.yml` добавлен сервис HAProxy:
```yaml
 haproxy:
   image: haproxy:3.0
   container_name: postgres_entrypoint
   ports:
     - 5435:5432
     - 6000:6000
   depends_on:
     - pg-master
     - pg-slave
     - zoo
   volumes:
     - ./haproxy:/usr/local/etc/haproxy
```

### Шаг 2. Создание конфигурации HAProxy
```
global
 maxconn 100

defaults
 log global
 mode tcp
 retries 3
 timeout client 30m
 timeout connect 4s
 timeout server 30m
 timeout check 5s

listen stats
 mode http
 bind *:7000
 stats enable
 stats uri /

listen postgres
 bind *:5432
 option httpchk
 http-check expect status 200
 default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
 server postgresql_pg_master_5432 pg-master:5432 maxconn 100 check port 8008
 server postgresql_pg_slave_5432 pg-slave:5432 maxconn 100 check port 8008
```

### Шаг 3. Перезапуск проекта

Перезапуск с новой конфигурацией:
```bash
docker-compose down
docker-compose up -d --build
```

### Шаг 4. Подключение через HAProxy

Подключение к БД через entrypoint:

Проверка работы репликации через entrypoint:

![Успешная вставка через entrypoint](./lab1/images/success.png)

**Результат:** Подключение успешно установлено, HAProxy автоматически перенаправляет запросы на мастер-ноду.

---

### Тест failover при отключении мастер-ноды

#### Шаг 1. Остановка мастер-ноды

Остановка контейнера с мастер-нодой:
```bash
docker stop pg-master
```

#### Шаг 2. Проверка работоспособности после failover

![Успешная вставка после остановки мастера](./lab1/images/success2.png)

**Результат:** Запись успешно выполнена. Новый мастер (бывший slave) принимает запросы на чтение и запись.

## Выводы

В ходе выполнения лабораторной работы:

1. **Успешно развернут** высокодоступный кластер PostgreSQL из двух нод с использованием Patroni
2. **Настроена потоковая репликация** - данные автоматически реплицируются с мастера на slave
3. **Реализована защита от записи** на slave-ноду (режим read-only)
4. **Настроен HAProxy** как единая точка входа для подключения к кластеру
5. **Протестирован механизм failover**:
   - При отказе мастер-ноды происходит автоматическое переключение на slave (~30 сек)
   - HAProxy автоматически перенаправляет трафик на новый мастер
   - Отсутствует потеря данных