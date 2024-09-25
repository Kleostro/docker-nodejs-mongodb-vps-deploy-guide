# Развертывание Node.js приложения с MongoDB в Docker на Timeweb VPS

Этот туториал описывает процесс развертывания Node.js приложения с использованием MongoDB в Docker-контейнере на виртуальном частном сервере (VPS) Timeweb.

## Предварительные требования

- Аккаунт на Timeweb
- VPS на Timeweb с установленной ОС (предпочтительно Ubuntu)
- Установленный Docker на вашем VPS
- Node.js приложение с MongoDB
- Репозиторий на GitHub с вашим приложением

## Шаг 1: Подготовка Dockerfile

1. В корневой директории вашего проекта создайте файл `Dockerfile`:

```Dockerfile
# Базовый образ Node.js
FROM node:18-alpine

# Создание рабочей директории в контейнере
WORKDIR /usr/src/app

# Копирование package.json и package-lock.json (если есть)
COPY package*.json ./

# Копирование файла prisma schema (если он существует)
COPY prisma ./prisma

# Установка зависимостей без выполнения скриптов
RUN npm install --ignore-scripts

# Генерация Prisma client
RUN npx prisma generate

# Копирование исходного кода приложения
COPY . .

# Сборка приложения
RUN npm run build

# Открытие порта
EXPOSE 3000

# Запуск приложения
CMD ["node", "dist/main"]
```

2. Создайте файл `.dockerignore`:

```
node_modules
npm-debug.log
```

## Шаг 2: Настройка GitHub Actions

1. В вашем репозитории создайте директорию `.github/workflows/`
2. Создайте файл `docker-publish.yml` в этой директории:

```yaml
name: Docker

on:
  push:
    branches: ["main"]
  pull_request:
    branches: ["main"]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push-image:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Log in to the Container registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata (tags, labels) for Docker
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

## Шаг 3: Настройка VPS на Timeweb

1. Подключитесь к вашему VPS через SSH:

   ```
   ssh root@your_server_ip
   ```

2. Установите Docker (если еще не установлен):

   ```bash
   sudo apt update
   sudo apt install apt-transport-https ca-certificates curl software-properties-common
   curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
   sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
   sudo apt update
   sudo apt install docker-ce
   ```

3. Проверьте установку Docker:
   ```bash
   sudo systemctl status docker
   ```

## Шаг 4: Развертывание приложения

1.  Загрузите и запустите Docker-контейнер:

    ```bash
    docker pull ghcr.io/your-username/your-repo:main
    docker run -d -p 3000:3000 -p 8877:8877 -e DATABASE_URL="your_mongodb_connection_string" ghcr.io/your-username/your-repo:main
    ```

    Замените `your_mongodb_connection_string` на реальный URL вашей MongoDB базы данных.

2.  Проверьте, что контейнер запущен:
    ```bash
    docker ps
    ```

## Шаг 5: Обновление приложения

Когда вы вносите изменения в ваш репозиторий и хотите обновить приложение на VPS, выполните следующие шаги:

1. Подключитесь к вашему VPS через SSH:

   ```
   ssh root@your_server_ip
   ```

2. Остановите текущий контейнер:

   ```bash
   docker stop $(docker ps -q --filter ancestor=ghcr.io/your-username/your-repo:main)
   ```

3. Удалите старый контейнер:

   ```bash
   docker rm $(docker ps -aq --filter ancestor=ghcr.io/your-username/your-repo:main)
   ```

4. Получите последнюю версию образа:

   ```bash
   docker pull ghcr.io/your-username/your-repo:main
   ```

5. Запустите новый контейнер:

   ```bash
   docker run -d -p 3000:3000 -e DATABASE_URL="your_mongodb_connection_string" ghcr.io/your-username/your-repo:main
   ```

6. Проверьте, что новый контейнер запущен:
   ```bash
   docker ps
   ```

Для автоматизации этого процесса вы можете создать скрипт обновления:

1. Создайте файл `update_container.sh`:

   ```bash
   #!/bin/bash
   docker stop $(docker ps -q --filter ancestor=ghcr.io/your-username/your-repo:main)
   docker rm $(docker ps -aq --filter ancestor=ghcr.io/your-username/your-repo:main)
   docker pull ghcr.io/your-username/your-repo:main
   docker run -d -p 3000:3000 -e DATABASE_URL="your_mongodb_connection_string" ghcr.io/your-username/your-repo:main
   ```

2. Сделайте скрипт исполняемым:

   ```bash
   chmod +x update_container.sh
   ```

3. Запустите скрипт для обновления:
   ```bash
   ./update_container.sh
   ```

## Шаг 6: Настройка Nginx (опционально)

Если вы хотите использовать доменное имя и HTTPS, вам нужно настроить Nginx как обратный прокси-сервер.

1. Установите Nginx:

   ```bash
   sudo apt install nginx
   ```

2. Создайте конфигурационный файл для вашего домена:

   ```bash
   sudo nano /etc/nginx/sites-available/your-domain.com
   ```

3. Добавьте следующую конфигурацию:

   ```nginx
    server {
        listen 80;
        server_name your-domain.com www.your-domain.com;

        location /api {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }

        location /docs {
            proxy_pass http://localhost:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
    }
   ```

## Шаг 7: Настройка SSL с Certbot (опционально)

Для обеспечения безопасности вашего приложения рекомендуется настроить HTTPS. Certbot позволяет легко получить и установить бесплатный SSL-сертификат от Let's Encrypt.

1. Установите Certbot и плагин для Nginx:

   ```bash
   sudo apt install certbot python3-certbot-nginx
   ```

2. Получите SSL-сертификат:

   ```bash
   sudo certbot --nginx -d your-domain.com -d www.your-domain.com
   ```

3. Следуйте инструкциям Certbot. Когда вас спросят о перенаправлении HTTP на HTTPS, рекомендуется выбрать опцию для автоматического перенаправления (обычно это опция 2).

4. После успешной установки Certbot автоматически обновит конфигурацию Nginx. Проверьте обновленный файл конфигурации:

   ```bash
   sudo nano /etc/nginx/sites-available/your-domain.com
   ```

5. Убедитесь, что конфигурация выглядит примерно так:

   ```nginx
   server {
       server_name your-domain.com www.your-domain.com;

       location /api {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }

       location /docs {
           proxy_pass http://localhost:3000;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }

       listen 443 ssl; # managed by Certbot
       ssl_certificate /etc/letsencrypt/live/your-domain.com/fullchain.pem; # managed by Certbot
       ssl_certificate_key /etc/letsencrypt/live/your-domain.com/privkey.pem; # managed by Certbot
       include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
       ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
   }

   server {
       if ($host = www.your-domain.com) {
           return 301 https://$host$request_uri;
       } # managed by Certbot

       if ($host = your-domain.com) {
           return 301 https://$host$request_uri;
       } # managed by Certbot

       listen 80;
       server_name your-domain.com www.your-domain.com;
       return 404; # managed by Certbot
   }
   ```

6. Если все выглядит правильно, перезапустите Nginx:
   ```bash
   sudo systemctl restart nginx
   ```

Теперь ваше приложение должно быть доступно через HTTPS:

- API: `https://your-domain.com/api`
- Документация API: `https://your-domain.com/docs`

Certbot автоматически настроит обновление сертификата. Вы можете проверить и обновить настройки автоматического обновления с помощью команды:

## Заключение

Теперь ваше Node.js приложение с MongoDB должно быть развернуто в Docker-контейнере на VPS Timeweb, доступно через настроенные порты 3000 и 8877, и легко обновляемо при внесении изменений в репозиторий.

Не забудьте регулярно проверять и обновлять ваше приложение, а также настроить мониторинг для обеспечения стабильной работы.
