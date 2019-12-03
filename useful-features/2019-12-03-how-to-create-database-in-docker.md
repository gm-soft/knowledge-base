---
category: useful features
tags: [docker, database, postgres]
---

# Как создать сервер базы данных в Docker

Шаг 1. Создать файл `docker-compose.yml`

```yml

version: '3.4'
services:
  database:
    image: centos/postgresql-10-centos7
    environment:
      POSTGRESQL_USER: user
      POSTGRESQL_PASSWORD: user
      POSTGRESQL_DATABASE: database_name
      POSTGRESQL_ADMIN_PASSWORD: user
    ports:
      - "5432:5432"

```

Шаг 2. Создать файл запуска bash-скрипта

```powershell

docker-compose -f "docker-compose.yml" stop
docker-compose -f "docker-compose.yml" rm --force
docker-compose -f "docker-compose.yml" up --build database

```
