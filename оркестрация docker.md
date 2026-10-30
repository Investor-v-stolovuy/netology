

---

## Задача 1

### Установка Docker и compose plugin

```bash
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER      # затем перелогиниться
docker --version
docker compose version
```

### index.html

```html
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```

### Dockerfile

```dockerfile
FROM nginx:1.29.0
COPY index.html /usr/share/nginx/html/index.html
```

### Сборка и публикация

```bash
docker pull nginx:1.29.0
docker build -t <username>/custom-nginx:1.0.0 .
docker login
docker push <username>/custom-nginx:1.0.0
```



---

## Задача 2

```bash
# запуск в фоне, порт опубликован только на localhost хоста
docker run -d --name ФИО-custom-nginx-t2 -p 127.0.0.1:8080:80 <username>/custom-nginx:1.0.0

# переименование без удаления
docker rename ФИО-custom-nginx-t2 custom-nginx-t2

# контрольная команда
date +"%d-%m-%Y %T.%N %Z" ; sleep 0.150 ; docker ps ; ss -tlpn | grep 127.0.0.1:8080 ; docker logs custom-nginx-t2 -n1 ; docker exec -it custom-nginx-t2 base64 /usr/share/nginx/html/index.html

# проверка доступности
curl http://127.0.0.1:8080
```

Ожидаемый ответ `curl`:

```html
<html>
<head>
Hey, Netology
</head>
<body>
<h1>I will be DevOps Engineer!</h1>
</body>
</html>
```


---

## Задача 3

### Подключение к потокам контейнера

```bash
docker attach custom-nginx-t2
# нажать Ctrl-P
docker ps -a
```


### Перезапуск и работа внутри

```bash
docker start custom-nginx-t2
docker exec -it custom-nginx-t2 bash
```

Внутри контейнера:

```bash
apt-get update && apt-get install -y nano
nano /etc/nginx/conf.d/default.conf     # listen 80; -> listen 81;
# либо: sed -i 's/listen\s*80;/listen 81;/' /etc/nginx/conf.d/default.conf

nginx -s reload
curl http://127.0.0.1:80     # curl: (7) Failed to connect ... Connection refused
curl http://127.0.0.1:81     # страница "Hey, Netology"
exit
```

### Проверка с хоста

```bash
ss -tlpn | grep 127.0.0.1:8080
docker port custom-nginx-t2
curl http://127.0.0.1:8080
```



### Удаление запущенного контейнера, не останавливая его

```bash
docker rm -f custom-nginx-t2
```

**Скриншот:** `<скриншот консоли>`

---

## Задача 4

```bash
# 1-й контейнер (centos), текущий каталог -> /data
docker run -d --name centos-t4 -v $(pwd):/data centos:7 sleep infinity

# 2-й контейнер (debian)
docker run -d --name debian-t4 -v $(pwd):/data debian sleep infinity

# файл из первого контейнера
docker exec -it centos-t4 bash -c 'echo "file from centos" > /data/from_centos.txt'

# файл на хосте
echo "file from host" > $(pwd)/from_host.txt

# листинг и содержимое во втором контейнере
docker exec -it debian-t4 bash -c 'ls -la /data; for f in /data/*.txt; do echo "== $f"; cat "$f"; done'
```


---

## Задача 5

### Подготовка

```bash
mkdir -p /tmp/netology/docker/task5 && cd /tmp/netology/docker/task5
```

`compose.yaml`:

```yaml
version: "3"
services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

`docker-compose.yaml`:

```yaml
version: "3"
services:
  registry:
    image: registry:2
    ports:
      - "5000:5000"
```

```bash
docker compose up -d
```

### Какой файл был запущен и почему

Запущен `compose.yaml` (сервис portainer). Согласно документации Docker, `compose.yaml` — предпочтительное (каноническое) имя файла. `docker-compose.yaml` поддерживается только для обратной совместимости. Если в каталоге есть оба файла, Compose использует `compose.yaml`, а второй игнорирует. Поэтому registry не стартовал.

> Атрибут `version` в современных версиях Compose устарел и игнорируется (будет предупреждение `version is obsolete`). Его можно убрать.

### Запуск обоих файлов через `include`

`compose.yaml`:

```yaml
version: "3"
include:
  - docker-compose.yaml
services:
  portainer:
    network_mode: host
    image: portainer/portainer-ce:latest
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

```bash
docker compose up -d
docker compose ps
```

### Загрузка образа в локальный registry

```bash
docker tag <username>/custom-nginx:1.0.0 127.0.0.1:5000/custom-nginx:latest
docker push 127.0.0.1:5000/custom-nginx:latest
curl http://127.0.0.1:5000/v2/_catalog       # {"repositories":["custom-nginx"]}
```

### Portainer

1. Открыть страницу Portainer и создать администратора (логин и пароль).
   > Portainer по умолчанию слушает `9000` по **HTTP**, а `9443` — по HTTPS. Если `https://127.0.0.1:9000` не открывается, используйте `http://127.0.0.1:9000`.
2. Перейти на `http://127.0.0.1:9000/#!/home` → выбрать окружение **local**.
3. **Stacks → Add stack → Web editor**, вставить:

```yaml
version: '3'

services:
  nginx:
    image: 127.0.0.1:5000/custom-nginx
    ports:
      - "9090:80"
```

4. Нажать **Deploy the stack**.
5. Открыть `http://127.0.0.1:9000/#!/2/docker/containers` → контейнер с nginx → **Inspect** → вид `<> Tree` → раскрыть `Config` → сделать скриншот от `AppArmorProfile` до `Driver`.

**Скриншоты:** `<Portainer со stack>`, `<Inspect: AppArmorProfile … Driver>`

### Удаление манифеста и предупреждение

```bash
rm compose.yaml
docker compose up -d
```

Предупреждение (пример):

```
WARN[0000] Found orphan containers ([task5-portainer-1]) for this project.
If you removed or renamed this service in your compose file, you can run this command with the --remove-orphans flag to clean it up.
```

**Суть предупреждения.** Compose определяет проект по имени каталога (`task5`). В этом проекте остался контейнер `portainer`, но сервиса `portainer` в оставшемся манифесте больше нет. Такой контейнер называется «осиротевшим» (orphan). Compose не знает, что с ним делать, и предлагает удалить его флагом `--remove-orphans`.

Выполняем предложенное действие:

```bash
docker compose up -d --remove-orphans
```

### Остановка проекта ОДНОЙ командой

```bash
docker compose down
```


