# Практическое занятие 15
## Шишков А.Д. ЭФМО-02-25
## Тема
Развертывание сервиса `tasks` на Linux VPS через `systemd`

## Цель
Освоить развертывание Go-сервиса на удаленном Linux-сервере, научиться запускать приложение как системную службу через `systemd`, выносить конфигурацию в отдельный `env`-файл, выполнять проверку работоспособности через `/health`, а также выполнять базовые операции сопровождения: запуск, автозапуск, просмотр логов, обновление и откат.

---

## Краткое описание выполненной работы

В рамках задания сервис `tasks` был собран в виде Linux-бинарника и развернут на VPS. Для запуска использовалась модель `binary + systemd`, без Docker. На сервере был создан отдельный системный пользователь `tasksuser`, подготовлены каталоги для бинарника и конфигурации, создан файл окружения `tasks.env`, а также unit-файл `tasks.service`.

После настройки служба была зарегистрирована в `systemd`, запущена и добавлена в автозапуск. Работоспособность проверена через:

- `systemctl status`
- `journalctl`
- `curl http://127.0.0.1:8082/health`

---

## Подготовка сервера

Сначала выполняется подключение к VPS по `SSH` и обновление пакетов:

```bash
ssh root@178.20.209.97
sudo apt update
sudo apt upgrade -y
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 1 - SSH-подключение к VPS]
```

Далее создается отдельный системный пользователь для запуска сервиса:

```bash
sudo useradd --system --no-create-home --shell /usr/sbin/nologin tasksuser
```

После этого создаются рабочие каталоги:

```bash
sudo mkdir -p /opt/tasks
sudo mkdir -p /etc/tasks
```

---

## Сборка Linux-бинарника

На локальной машине сервис `tasks` собирается в Linux-бинарник:

```bash
mkdir -p bin
GOOS=linux GOARCH=amd64 go build -o bin/tasks ./services/tasks/cmd/tasks
```

После сборки получается исполняемый файл `tasks`, предназначенный для запуска на Linux VPS.

---

## Копирование бинарника на сервер

Собранный бинарник копируется на VPS:

```bash
scp bin/tasks root@178.20.209.97:/root/tasks
```

Затем на сервере файл переносится в рабочую директорию сервиса и получает нужные права:

```bash
sudo mv /root/tasks /opt/tasks/tasks
sudo chown tasksuser:tasksuser /opt/tasks/tasks
sudo chmod 755 /opt/tasks/tasks
```

---

## Конфигурационный файл

Файл окружения создается по пути:

```bash
sudo nano /etc/tasks/tasks.env
```

Содержимое файла:

```env
TASKS_PORT=8082
DATABASE_URL=postgres://tasks:tasks@127.0.0.1:5432/tasks?sslmode=disable
AUTH_MODE=http
AUTH_BASE_URL=http://127.0.0.1:8081
REDIS_ADDR=127.0.0.1:6379
RABBIT_URL=amqp://guest:guest@127.0.0.1:5672/
JOB_QUEUE_NAME=task_jobs
JOB_DLQ_NAME=task_jobs_dlq
EVENT_QUEUE_NAME=task_events
```

После создания конфигурационного файла задаются права доступа:

```bash
sudo chown root:root /etc/tasks/tasks.env
sudo chmod 600 /etc/tasks/tasks.env
```

---

## Unit-файл systemd

Для запуска приложения как системной службы создается unit-файл:

```bash
sudo nano /etc/systemd/system/tasks.service
```

Содержимое unit-файла:

```ini
[Unit]
Description=Tasks Service
After=network.target

[Service]
Type=simple
User=tasksuser
WorkingDirectory=/opt/tasks
EnvironmentFile=/etc/tasks/tasks.env
ExecStart=/opt/tasks/tasks
Restart=always
RestartSec=2
NoNewPrivileges=true
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

---

## Запуск службы

После создания unit-файла выполняется перечитывание конфигурации `systemd`, запуск службы и включение автозапуска:

```bash
sudo systemctl daemon-reload
sudo systemctl start tasks
sudo systemctl enable tasks
```

Проверка статуса службы:

```bash
sudo systemctl status tasks
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 2 - Статус службы tasks]
```

Проверка автозапуска:

```bash
sudo systemctl is-enabled tasks
```

Ожидаемый результат:

```text
enabled
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 3 - Автозапуск службы]
```

---

## Просмотр логов

Для просмотра последних записей журнала используется команда:

```bash
sudo journalctl -u tasks --no-pager -n 50
```

Для просмотра логов в реальном времени:

```bash
sudo journalctl -u tasks -f
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 4 - Логи службы через journalctl]
```

---

## Проверка работоспособности

После запуска службы выполняется проверка health endpoint на самом сервере:

```bash
curl -i http://127.0.0.1:8082/health
```

Ожидаемый ответ:

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"instance":"tasks-default","service":"tasks","status":"ok"}
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 5 - Проверка /health]
```

---

## Обновление приложения

Обновление сервиса выполняется заменой бинарника и перезапуском службы.

На локальной машине собирается новая версия:

```bash
GOOS=linux GOARCH=amd64 go build -o bin/tasks ./services/tasks/cmd/tasks
scp bin/tasks root@178.20.209.97:/root/tasks.new
```

На сервере:

```bash
sudo cp /opt/tasks/tasks /opt/tasks/tasks.bak
sudo mv /root/tasks.new /opt/tasks/tasks
sudo chown tasksuser:tasksuser /opt/tasks/tasks
sudo chmod 755 /opt/tasks/tasks
sudo systemctl restart tasks
sudo systemctl status tasks
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 6 - Обновление службы]
```

---

## Откат к предыдущей версии

Если после обновления новая версия работает некорректно, можно вернуть предыдущий бинарник:

```bash
sudo mv /opt/tasks/tasks.bak /opt/tasks/tasks
sudo chown tasksuser:tasksuser /opt/tasks/tasks
sudo chmod 755 /opt/tasks/tasks
sudo systemctl restart tasks
sudo systemctl status tasks
```

```text
[МЕСТО ДЛЯ СКРИНШОТА 7 - Откат службы]
```

---
## Выводы

- Сервис `tasks` успешно развернут на Linux VPS в виде отдельного бинарника.
- Для запуска использован `systemd`, что обеспечивает удобное управление службой и автозапуск.
- Конфигурация вынесена в отдельный файл окружения, что упрощает сопровождение.
- Работоспособность сервиса подтверждена через `systemctl`, `journalctl` и `/health`.
- Подготовлены базовые процедуры обновления и отката, необходимые для эксплуатации приложения.
