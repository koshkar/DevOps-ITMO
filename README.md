# Лабораторная работа №3

Работы выполнили: Команда "Буцефалы"

## Введение

Для начала обусловимся, что в качестве инструмента CI/CD мы будем использовать ***Gitlab CI***.
Определимся с тем, из чего вообще должен состоять наш ***.gitlab-ci.yml*** и как он выполняется.

В общем можно сказать, что пайплайн будет состоять из нескольких этапов - это *build*, *test* и *deploy*. На каждый этап у нас есть джобы - набор скриптов для выполнения задач. Далее идет настройка окружения для использования переменных. Далее можем подключить кэширование для оптимизации, задавать какие-либо условия для выполнения ну и добавить деплой для автоматического развертывания. Итак, начнем-с.

## Bad practices

```
stages:
  - build
  - test
  - deploy

variables:
  DEPLOY_ENV: production

build_job:
  stage: build
  script:
    - echo "Building the app..."
    - npm install
    - npm run build
  only:
    - main

test_job:
  stage: test
  script:
    - echo "Running tests..."
    - npm install
    - npm test
  only:
    - main

deploy_job:
  stage: deploy
  script:
    - echo "Deploying to production..."
    - ssh user@production-server 'cd /app && git pull && npm install && pm2 restart app'
  only:
    - main
    ```
Определим ряд ~~не~~ очевидных проблем.

1. Повторяющаяся установка зависимостей `npm install`.
2. Отсутствие окружений для тестирования. (Нет staging).
3. Прям в коде шарашим SSH-доступ с логином и паролем, а это небезопасно.
4. Не используем кэширование, то есть зависимости будут постоянно накатываться.
5. Ну и деплой напрямую из main ветки, что означает, что при любом изменении в ветке происходит автодеплой без ручной проверки.

## Good practices

Что ж, а теперь самое время исправить наши косяки:
```
stages:
  - build
  - test
  - deploy_staging
  - deploy_production

cache:
  paths:
    - node_modules/

variables:
  STAGING_ENV: staging
  PRODUCTION_ENV: production

build_job:
  stage: build
  script:
    - echo "Building the app..."
    - npm ci
    - npm run build
  only:
    - main

test_job:
  stage: test
  script:
    - echo "Running tests..."
    - npm ci
    - npm test
  only:
    - main

deploy_staging_job:
  stage: deploy_staging
  script:
    - echo "Deploying to staging..."
    - ssh user@staging-server 'cd /app && git pull && npm ci && pm2 restart app'
  only:
    - main

deploy_production_job:
  stage: deploy_production
  script:
    - echo "Deploying to production..."
    - ssh user@production-server 'cd /app && git pull && npm ci && pm2 restart app'
  only:
    - manual

```
На что же теперь стоит обратить внимание:
1. Изменили `npm install` на `npm ci`. Это гораздо быстрее и предназначено для CI. Также внедрили кэширование, теперь повторные запуски куда быстрее.
2. Добавили стадию *deploy_staging*, что позволяет нам развернуть приложение в тестовое окружение перед prod.
3. Хоть напрямую из примера не видно, но чувствительные данные вынесены в переменные окружения.
4. Теперь деплой происходит вручную при помощи опции *manual*.

## Вывод
В итоге, как наши изменения повлияли на работу:
Теперь процесс CI/CD стал более безопасным, быстрым и контролируемым. Каждый из этих пунктов очень важен для успешного развертования приложений в prod. Все как завещал дядюшка Боб.