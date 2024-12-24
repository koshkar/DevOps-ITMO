# Лабораторная работа №3

Работы выполнили: Команда "Буцефалы"

## Введение

В данной лабораторной работе мы рассмотрим настройку CI/CD пайплайна с использованием GitHub Actions. 
Наш пайплайн будет состоять из нескольких этапов: build, test, deploy_staging и deploy_production. На каждом этапе будут выполняться определенные джобы, такие как установка зависимостей, сборка приложения, запуск тестов и деплой в различные окружения. Также мы рассмотрим настройку переменных окружения, кэширование для оптимизации сборки, и тд.

## Bad practices

```
name: CI Pipeline

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Install dependencies
        run: npm install

      - name: Build the app
        run: npm run build

  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Deploy to Production
        run: ssh user@production-server 'cd /app && git pull && npm install && pm2 restart app'
```

Определим ряд ~~не~~ очевидных проблем.

1. Повторяющаяся установка зависимостей npm install: Зависимости устанавливаются отдельно, что увеличивает время выполнения пайплайна.
2. Отсутствие отдельных окружений для тестирования и прода: Все деплои выполняются напрямую в прод без промежуточного тестирования.
3. Небезопасное хранение SSH-доступа: Прямое использование SSH-команд в скриптах без защиты.
4. Отсутствие кэширования зависимостей: Зависимости устанавливаются заново при каждом запуске, что замедляет процесс.
5. Автодеплой из основной ветки: Любые изменения в основной ветке сразу деплоятся в продакшен без ручной проверки.

## Good practices

Что ж, а теперь самое время исправить наши косяки:

```
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
      - lab-3
  pull_request:
    branches:
      - main
  workflow_dispatch: # Позволяет запускать workflow вручную

env:
  STAGING_ENV: staging
  PRODUCTION_ENV: production

jobs:
  build:
    name: Build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/lab-3'
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        run: npm ci

      - name: Build the app
        run: npm run build

  test:
    name: Test
    runs-on: ubuntu-latest
    needs: build
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Cache node modules
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install dependencies
        run: npm ci

      - name: Run tests
        run: npm test

  deploy_staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.5.4
        with:
          ssh-private-key: ${{ secrets.STAGING_SSH_KEY }}

      - name: Deploy to Staging Server
        run: |
          ssh user@staging-server 'cd /app && git pull && npm ci && pm2 restart app'

  deploy_production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy_staging
    if: github.event_name == 'workflow_dispatch'
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.5.4
        with:
          ssh-private-key: ${{ secrets.PRODUCTION_SSH_KEY }}

      - name: Deploy to Production Server
        run: |
          ssh user@production-server 'cd /app && git pull && npm ci && pm2 restart app'
```

На что же теперь стоит обратить внимание:
1. Изменили npm install на npm ci. Это гораздо быстрее и предназначено для CI. Также внедрили кэширование, теперь повторные запуски куда быстрее.
2. Добавили стадию deploy_staging, что позволяет нам развернуть приложение в тестовое окружение перед prod.
3. Чувствительные данные вынесены в переменные окружения. SSH-ключи хранятся в секретах GitHub, обеспечивая безопасность доступа.
4. Теперь деплой происходит вручную при помощи опции workflow_dispatch. Это позволяет контролировать момент деплоя в production и проводить дополнительную проверку перед развертыванием.

## Вывод
В итоге, как наши изменения повлияли на работу:
Теперь процесс CI/CD стал более безопасным, быстрым и контролируемым. Каждый из этих пунктов очень важен для успешного развертования приложений в prod. Все как завещал дядюшка Боб.
