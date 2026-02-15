# Установка на чистый сервер (Ubuntu/Debian)

## Требования

- **ОС**: Ubuntu 20.04+ или Debian 11+
- **RAM**: минимум 2 GB (рекомендуется 4+ GB)
- **Disk**: минимум 10 GB свободного места
- **Доступ**: root или sudo

---

## Шаг 1: Подготовка файлов

### На текущем сервере — перенести образ

```bash
# Если файл ещё не создан, создаём:
docker save cc-course:latest | gzip > cc-course-latest.tar.gz

# Переносим на новый сервер (замените user@new-server):
scp /root/cc-course-latest.tar.gz user@new-server:/root/
scp .env.example user@new-server:/root/
```

---

## Шаг 2: Установка Docker на чистом сервере

### На новом сервере:

```bash
# Обновление системы
sudo apt-get update && sudo apt-get upgrade -y

# Установка Docker
curl -fsSL https://get.docker.com | sh

# Добавление пользователя в группу docker (опционально)
sudo usermod -aG docker $USER
newgrp docker

# Проверка
docker --version
```

---

## Шаг 3: Загрузка образа

```bash
cd /root

# Загрузка образа (займет 1-2 минуты)
gunzip -c cc-course-latest.tar.gz | docker load

# Проверка что образ загрузился
docker images | grep cc-course
# Ожидаемый вывод: cc-course   latest   8a886fc6749a   ...
```

---

## Шаг 4: Настройка .env файла

```bash
# Создать .env из шаблона
cp .env.example .env

# Редактировать
nano .env
```

### Минимальная настройка `.env`:

```bash
# Обязательные параметры
GLM_API_KEY=ваш_ключ_от_z_ai        # Получить на https://open.bigmodel.cn/
PASSWORD=student-2026                 # Пароль для входа в систему

# Опционально (если нужно изменить порт)
# PORT=8080                           # Порт по умолчанию
```

### Получение API ключа Z.AI:

1. Перейдите на https://open.bigmodel.cn/
2. Зарегистрируйтесь / войдите
3. Создайте API ключ
4. Скопируйте его в `GLM_API_KEY`

---

## Шаг 5: Запуск контейнера

### Вариант A: Простой запуск (через docker run)

```bash
docker run -d \
  --name cc-course \
  --restart unless-stopped \
  -p 8080:8080 \
  -v course-data:/home/coder/course \
  --env-file .env \
  cc-course:latest
```

### Вариант B: Через docker-compose (рекомендуется)

```bash
# Создать docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  student:
    image: cc-course:latest
    container_name: cc-course
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - course-data:/home/coder/course
    environment:
      - PASSWORD=${PASSWORD:-student-2026}
      - GLM_API_KEY=${GLM_API_KEY}
      - GLM_API_KEY_BACKUP=${GLM_API_KEY_BACKUP:-}
      - ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL:-https://api.z.ai/api/anthropic}
      - ANTHROPIC_DEFAULT_OPUS_MODEL=${ANTHROPIC_DEFAULT_OPUS_MODEL:-GLM-5}
      - ANTHROPIC_DEFAULT_SONNET_MODEL=${ANTHROPIC_DEFAULT_SONNET_MODEL:-GLM-5}
      - ANTHROPIC_DEFAULT_HAIKU_MODEL=${ANTHROPIC_DEFAULT_HAIKU_MODEL:-GLM-4.5-Air}
      - API_TIMEOUT_MS=${API_TIMEOUT_MS:-3000000}

volumes:
  course-data:
EOF

# Запуск
docker compose up -d
# или для старых версий: docker-compose up -d
```

---

## Шаг 6: Проверка работы

```bash
# Проверить что контейнер запущен
docker ps

# Посмотреть логи
docker logs cc-course

# Проверить здоровье контейнера
docker inspect cc-course | grep Health -A 5
```

Ожидаемый статус: `Up` и `(healthy)` или `health: starting`

---

## Шаг 7: Firewall (если включён)

### Ubuntu (UFW):

```bash
sudo ufw allow 8080/tcp
sudo ufw reload
```

### CentOS/RHEL (firewalld):

```bash
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```

### Облачные провайдеры:

Не забудьте открыть порт **8080** в security group вашего облачного провайдера!

---

## Доступ к системе

После успешного запуска:

1. Откройте в браузере: `http://ваш-ip-адрес:8080`
2. Введите пароль из `.env` (по умолчанию: `student-2026`)

### Доступные разделы:

| Путь | Что это |
|------|---------|
| `/ide/` | VS Code в браузере |
| `/files/` | Файловый менеджер |
| `/healthz` | Health check |

---

## Управление контейнером

```bash
# Остановить
docker stop cc-course

# Запустить
docker start cc-course

# Перезапустить
docker restart cc-course

# Посмотреть логи в реальном времени
docker logs -f cc-course

# Удалить контейнер (данные сохранятся в volume)
docker rm -f cc-course

# Удалить контейнер и данные
docker rm -f cc-course
docker volume rm course-data
```

---

## Изменение API ключа

### Способ 1: Через .env и пересоздание

```bash
# Редактировать .env
nano .env

# Пересоздать контейнер
docker compose down
docker compose up -d
```

### Способ 2: Внутри работающего контейнера

```bash
# Выполнить команду внутри контейнера
docker exec -it cc-course ./switch-api-key.sh backup
# или
docker exec -it cc-course ./switch-api-key.sh primary
```

---

## Как не потерять контейнер

> **Главное правило:** `docker compose up` пересоздаёт контейнер при любом изменении конфига. Старый контейнер удаляется автоматически.

### Что НЕ делать

```bash
# ОПАСНО: если вы поменяли порт/имя в docker-compose.yml и запустили —
# старый контейнер будет УДАЛЁН и заменён новым
docker compose up -d
```

### Безопасная остановка / запуск (без потери данных)

```bash
# Остановить контейнер (данные сохраняются)
docker stop cc-course

# Запустить обратно
docker start cc-course

# Перезапустить (например после зависания)
docker restart cc-course
```

### Как запустить второй контейнер рядом с первым

Если нужно два контейнера на разных портах — **не меняйте** существующий `docker-compose.yml`. Вместо этого:

```bash
# 1. Скопируйте всю директорию
cp -r cc-for-non-coders-dev-container cc-for-non-coders-dev-container-2
cd cc-for-non-coders-dev-container-2

# 2. Измените в docker-compose.yml три вещи:
#    - container_name: другое имя (например claude-course-2)
#    - ports: другой порт (например 8082:8080)
#    - volumes: другой volume (например student-data-2)

# 3. Скопируйте .env
cp ../cc-for-non-coders-dev-container/.env .env

# 4. Запустите — это НЕ затронет первый контейнер
docker compose up -d
```

### Бэкап данных студента

```bash
# Создать бэкап volume в tar-архив
docker run --rm -v course-data:/data -v $(pwd):/backup \
  alpine tar czf /backup/course-backup-$(date +%Y%m%d).tar.gz -C /data .

# Восстановить из бэкапа
docker run --rm -v course-data:/data -v $(pwd):/backup \
  alpine sh -c "cd /data && tar xzf /backup/course-backup-XXXXXXXX.tar.gz"
```

### Где хранятся данные

| Что | Где | Переживает пересоздание контейнера? |
|-----|-----|-------------------------------------|
| Файлы студента (`/home/coder/course/`) | Docker volume `course-data` | Да |
| Настройки Claude Code | Внутри контейнера | Нет — создаются заново из `.env` |
| Образ Docker | `docker images` | Да, пока не удалите вручную |

> **Вывод:** данные студента в volume не теряются при пересоздании контейнера. Но сам контейнер (имя, порт, настройки) пересоздаётся с нуля.

---

## Troubleshooting

### Контейнер не запускается

```bash
# Посмотреть логи
docker logs cc-course

# Проверить не занят ли порт
sudo lsof -i :8080
# или
sudo netstat -tlnp | grep 8080
```

### Ошибка "address already in use"

```bash
# Найти процесс занимающий порт
sudo ss -tlnp | grep 8080

# Использовать другой порт, например 8081:
docker run -d --name cc-course -p 8081:8080 ...
```

### API ключ не работает

```bash
# Проверить что ключ установлен в контейнере
docker exec cc-course cat /home/coder/.claude/.env | grep ANTHROPIC_AUTH_TOKEN

# Проверить формат ключа (должен быть вида xxxxxxxxxxx.xxx)
```

### Не хватает памяти

```bash
# Проверить свободную память
free -h

# Если мало RAM, добавить swap (например 2GB):
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

---

## Быстрый старт (копипаст для чистого сервера)

```bash
# 1. Установить Docker
curl -fsSL https://get.docker.com | sh

# 2. Загрузить образ (предварительно скопировав файл)
gunzip -c cc-course-latest.tar.gz | docker load

# 3. Создать .env (заменить YOUR_API_KEY)
cat > .env << EOF
GLM_API_KEY=YOUR_API_KEY
PASSWORD=student-2026
EOF

# 4. Запустить
docker run -d \
  --name cc-course \
  --restart unless-stopped \
  -p 8080:8080 \
  -v course-data:/home/coder/course \
  --env-file .env \
  cc-course:latest

# 5. Проверить
sleep 5 && docker logs cc-course | tail -10
```

---

## Поддержка

- Проект: https://github.com/miolamio/cc-for-non-coders-dev-container
- API ключи: https://open.bigmodel.cn/
