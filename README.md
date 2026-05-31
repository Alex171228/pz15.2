# Практическое занятие 15
## Шишков А.Д. ЭФМО-02-25
## Тема
Деплой приложения на VPS. Настройка systemd

## Цель
Освоить публикацию backend-приложения на удалённом Linux-сервере, научиться подключаться к VPS по SSH, размещать исполняемый файл приложения, настраивать переменные окружения, создавать unit-файл systemd, управлять сервисом через systemctl, анализировать логи через journalctl и выполнять базовую процедуру обновления версии приложения.

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

<img width="891" height="530" alt="image" src="https://github.com/user-attachments/assets/1de0d4a1-2cca-4882-80bb-92b97d3d24c3" /> 


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

<img width="2152" height="462" alt="image" src="https://github.com/user-attachments/assets/781c2ecc-d31a-4e72-87ea-f423676065f0" /> 


Проверка автозапуска:

```bash
sudo systemctl is-enabled tasks
```

Ожидаемый результат:

```text
enabled
```

<img width="552" height="51" alt="image" src="https://github.com/user-attachments/assets/f3e67970-2738-486f-b3c0-9221611528b4" /> 


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

<img width="2177" height="293" alt="image" src="https://github.com/user-attachments/assets/a883e062-0a8a-4aa9-9a66-66ecc6f08b4e" /> 


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

<img width="623" height="191" alt="image" src="https://github.com/user-attachments/assets/9e67e85a-1757-4a94-9192-084d9e5d4c4c" /> 


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

<img width="2141" height="644" alt="image" src="https://github.com/user-attachments/assets/96986f66-301f-492d-b62b-97729fdd0c88" /> 


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

<img width="1805" height="392" alt="image" src="https://github.com/user-attachments/assets/94e9a213-cf74-4c3c-937f-1aaecd35c979" />

---
## Выводы

- Сервис `tasks` успешно развернут на Linux VPS в виде отдельного бинарника.
- Для запуска использован `systemd`, что обеспечивает удобное управление службой и автозапуск.
- Конфигурация вынесена в отдельный файл окружения, что упрощает сопровождение.
- Работоспособность сервиса подтверждена через `systemctl`, `journalctl` и `/health`.
- Подготовлены базовые процедуры обновления и отката, необходимые для эксплуатации приложения.
