# DevOps-ITMO

# Лабораторная работа №3*

Работы выполнили: Команда "Буцефалы"

## Введение

В данной лабораторной работе мы рассмотрим работу с секретами в CI/CD пайплайне. 
Разберём плохую практику работы с секретами, приведём к ней пример, а затем исправим, чтобы показать, как правильно организовать их безопасное использование. 

## Bad practices

```
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js (example for Node.js project)
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

      - name: Build project
        run: npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install Doppler CLI
        uses: dopplerhq/cli-action@v1
        with:
          token: ${{ secrets.DOPPLER_TOKEN }}

      - name: Load Secrets into Environment
        env:
          DOPPLER_TOKEN: ${{ secrets.DOPPLER_TOKEN }}
        run: |
          doppler secrets download --no-file --format env > secrets.env
          set -o allexport
          source secrets.env
          set +o allexport

      - name: Use Secrets Insecurely
        run: |
          echo "Building application with a visible secret..."

          echo "SECRET_KEY: $SECRET_KEY"  # Небезопасно: ключ отображается в логах!

          export DATABASE_URL="postgresql://user:$SECRET_KEY@localhost:5432/mydatabase"
          echo "Connecting to the database using DATABASE_URL: $DATABASE_URL"  # Небезопасно: полный URL в логах!

          if [ -f "./run_database_migrations.sh" ]; then
            ./run_database_migrations.sh --db-url=$DATABASE_URL
          else
            echo "Migration script not found. Skipping migrations."
          fi
        env:
          SECRET_KEY: $SECRET_KEY
```

Проблемы:

Секреты светятся в логах
Использование echo для отображения значений переменных (например, SECRET_KEY и DATABASE_URL) в логах делает их доступными для любого, кто имеет доступ к логам пайплайна.
Секреты могут быть случайно использованы в неправильных местах
Передача секретов напрямую в логи приводит к возможной утечке, если доступ к логам окажется скомпрометированным.
Секреты остаются видимыми в пайплайне
Такой подход нарушает принцип минимизации утечек и использования секретов только в безопасных местах.

![Bad secret](visiblesec.png)

## Good practices

Что ж, а теперь самое время исправить наши косяки:

```
name: CI/CD Pipeline

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Node.js (example for Node.js project)
        uses: actions/setup-node@v3
        with:
          node-version: '16'

      - name: Install dependencies
        run: npm install

      - name: Run tests
        run: npm test

      - name: Build project
        run: npm run build

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Install Doppler CLI
        uses: dopplerhq/cli-action@v1
        with:
          token: ${{ secrets.DOPPLER_TOKEN }}

      - name: Load Secrets into Environment
        env:
          DOPPLER_TOKEN: ${{ secrets.DOPPLER_TOKEN }}
        run: |
          doppler secrets download --no-file --format env > secrets.env
          set -o allexport
          source secrets.env
          set +o allexport

      - name: Use Secrets Securely
        run: |
          echo "Building application with a hidden secret..."

          export DATABASE_URL="postgresql://user:$SECRET_KEY@localhost:5432/mydatabase"
          echo "Connecting to the database using the secret!"

          if [ -f "./run_database_migrations.sh" ]; then
            ./run_database_migrations.sh --db-url=$DATABASE_URL
          else
            echo "Migration script not found. Skipping migrations."
          fi
        env:
          SECRET_KEY: $SECRET_KEY
```

Преимущества:
Секреты не отображаются в логах
Все конфиденциальные данные скрыты и не выводятся в консоль или логи.
Централизованное управление секретами
Использование Doppler позволяет безопасно управлять секретами через внешнее хранилище, минимизируя риск утечек.
Гибкость и безопасность
Даже если лог-файлы будут доступны третьим лицам, никакая конфиденциальная информация не будет скомпрометирована.
Секреты используются только в безопасной среде
Загружаются и обрабатываются только в процессе пайплайна без их сохранения в коде или на сервере.

![Good secret](invisiblesec.png)

## Приложения
Результат пайплайнов плохого и хорошего секретов (можно подробнее ознакомиться в репо в Actions):
![Bad pipe](badsec.png)
![Good pipe](goodsec.png)

## Вывод
Безопасное использование секретов в CI/CD пайплайнах возможно только при условии исключения их отображения в логах, применения внешних хранилищ секретов (например, Doppler) и централизованного управления. 
Эти подходы позволяют минимизировать риски утечек и сделать пайплайн надёжным и безопасным.

Команда Буцефалы

![Логооооо](bucefalu.jpeg)
