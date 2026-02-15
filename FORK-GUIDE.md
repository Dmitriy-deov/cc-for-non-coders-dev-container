# Создание форка и внесение изменений

## Шаг 1: Создайте форк на GitHub

1. Откройте в браузере: https://github.com/miolamio/cc-for-non-coders-dev-container

2. Нажмите кнопку **Fork** (вверху справа)

3. После создания форка у вас будет:
   `https://github.com/ВАШ_ЛОГИН/cc-for-non-coders-dev-container`

---

## Шаг 2: Скопируйте файлы на ваш компьютер

Сохраните файлы из этого архива на ваш компьютер:
- `entrypoint.sh` — исправленный скрипт запуска
- `INSTALL-FRESH-SERVER.md` — инструкция по установке на чистый сервер
- `FORK-GUIDE.md` — этот файл

Или скачайте весь репозиторий как zip.

---

## Шаг 3: Клонируйте ВАШ форк на компьютер

Откройте терминал (или Git Bash на Windows) и выполните:

```bash
# Замените ВАШ_ЛОГИН на ваш реальный логин GitHub
git clone https://github.com/ВАШ_ЛОГИН/cc-for-non-coders-dev-container.git
cd cc-for-non-coders-dev-container
```

---

## Шаг 4: Замените файл исправленной версией

### Способ A: Заменить entrypoint.sh (Windows)

```bash
# Скопируйте исправленный entrypoint.sh в папку с репозиторием
# Замените существующий файл
```

### Способ B: Применить патч (Linux/Mac/Git Bash)

```bash
# Применить изменения автоматически
patch -p1 < entrypoint-fix.patch
```

---

## Шаг 5: Закоммитить изменения

```bash
git add entrypoint.sh INSTALL-FRESH-SERVER.md FORK-GUIDE.md
git commit -m "Fix entrypoint.sh for volume permissions + add installation guide"
```

---

## Шаг 6: Отправить в ваш форк (push)

```bash
git push origin main
```

---

## Шаг 7: Готово!

Теперь на любом новом сервере можно установить из вашего форка:

```bash
git clone https://github.com/ВАШ_ЛОГИН/cc-for-non-coders-dev-container.git
cd cc-for-non-coders-dev-container
cp .env.example .env
nano .env  # ввести API ключ
docker compose build
docker compose up -d
```

---

## Что было изменено

### entrypoint.sh (строка 17)

**Было:**
```bash
cp -a /home/coder/.course-image/. /home/coder/course/
```

**Стало:**
```bash
cp -r --no-preserve=mode,ownership /home/coder/.course-image/. /home/coder/course/ 2>/dev/null || cp -r /home/coder/.course-image/. /home/coder/course/
```

**Зачем:** Оригинальная команда `cp -a` пытается сохранить права доступа и временные метки, что вызывает ошибку при работе с Docker volume. Исправленная версия игнорирует эти атрибуты и работает без ошибок.

---

## Структура файлов после форка

```
cc-for-non-coders-dev-container/
├── Dockerfile              # unchanged
├── docker-compose.yml      # unchanged
├── entrypoint.sh           # ИСПРАВЛЕН
├── auth-gateway.py         # unchanged
├── login.html              # unchanged
├── .env.example            # unchanged
├── INSTALL-FRESH-SERVER.md # НОВЫЙ
├── FORK-GUIDE.md           # НОВЫЙ
├── course/                 # unchanged
└── skills/                 # unchanged
```
