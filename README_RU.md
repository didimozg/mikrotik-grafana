
# 🛡️ Инструкция: Карта кибератак с MikroTik в Grafana

English documentation: [README.md](./README.md).

**Стек:** MikroTik → Rsyslog (LXC) → Promtail(LXC) → Loki(LXC) → Grafana.

-----

## Этап 1. Настройка сервера (Proxmox LXC)

Мы используем **Rsyslog** как надежный буфер, чтобы избежать проблем с форматами логов MikroTik.

### 1\. Подготовка LXC контейнера

Зайдите в консоль вашего контейнера (где будут стоять Rsyslog и Promtail).

**Установите Rsyslog (если нет):**

```bash
apt update && apt install -y rsyslog curl unzip
```

**Настройте Rsyslog для приема UDP:**
Откройте конфиг `/etc/rsyslog.conf`:

```bash
nano /etc/rsyslog.conf
```

Раскомментируйте эти строки (уберите `#`):

```conf
module(load="imudp")
input(type="imudp" port="1514")
```

*(Опционально: закомментируйте `module(load="imklog")`, чтобы не было ошибок прав доступа).*

**Настройте фильтрацию в отдельный файл:**
Создайте файл `/etc/rsyslog.d/mikrotik.conf`:

```bash
nano /etc/rsyslog.d/mikrotik.conf
```

Вставьте (замените `192.168.X.X` на IP вашего Микротика):

```conf
if $fromhost-ip == '192.168.X.X' then {
    action(type="omfile" file="/var/log/mikrotik.log")
    stop
}
```

**Настройте ротацию логов (чтобы диск не переполнился):**
Создайте файл `/etc/logrotate.d/mikrotik`:

```bash
nano /etc/logrotate.d/mikrotik
```

Вставьте:

```conf
/var/log/mikrotik.log {
    daily
    rotate 7
    size 100M
    compress
    delaycompress
    missingok
    notifempty
    postrotate
        /usr/lib/rsyslog/rsyslog-rotate
    endscript
}
```

**Перезапустите Rsyslog:**

```bash
systemctl restart rsyslog
```

-----

## Этап 2. Настройка Promtail

Promtail будет читать файл `/var/log/mikrotik.log`, определять страну по IP и отправлять в Loki.

### 1\. Скачивание GeoIP базы

Вам нужен файл `GeoLite2-City.mmdb`.

```bash
wget https://github.com/P3TERX/GeoLite.mmdb/releases/download/2025.11.25/GeoLite2-City.mmdb
```
### 2\. Конфигурация Promtail

Создайте/отредактируйте `/etc/promtail/config.yaml`:

```bash
nano /etc/promtail/config.yaml
```

Вставьте этот код (убедитесь, что `clients: url` указывает на ваш Loki):

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /var/lib/promtail/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: "mikrotik_via_rsyslog"
          __path__: /var/log/mikrotik.log

    pipeline_stages:
      # 1. Попытка №1: Ищем формат Firewall (IP:PORT->)
      - regex:
          expression: ',\s(?P<source_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}):\d+->'

      # 2. Попытка №2: Ищем классический формат (src-address=)
      - regex:
          expression: 'src-address=(?P<source_ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})'

      # 3. Определяем страну и город
      - geoip:
          db: "/etc/promtail/GeoLite2-City.mmdb"
          source: "source_ip"
          db_type: "city"

      # 4. Упаковываем данные для отправки
      - pack:
          labels:
            - source_ip
            - geoip_country_name
            - geoip_city_name
            - geoip_location_latitude
            - geoip_location_longitude
```

**Создайте папку для позиций и перезапустите:**

```bash
mkdir -p /var/lib/promtail
systemctl restart promtail
```

-----

## Этап 3. Настройка MikroTik

Настроим роутер на отправку логов на наш сервер.

### 1\. Создание Action (Куда слать)

Выполните в терминале MikroTik (замените `192.168.X.Y` на IP вашего LXC):

```mikrotik
/system logging action add \
    name=lokisyslog \
    target=remote \
    remote=192.168.X.Y \
    remote-port=1514 \
    src-address=0.0.0.0 \
    remote-log-format=bsd-syslog \
    syslog-time-format=bsd-syslog \
    syslog-facility=local0
```

*(Примечание: Если v7 ругается на `bsd-syslog`, используйте `remote-log-format=default`)*.

### 2\. Настройка правил (Что слать)

```mikrotik
/system logging add topics=info action=lokisyslog
/system logging add topics=error action=lokisyslog
/system logging add topics=firewall action=lokisyslog
```

### 3\. Включение логов Firewall (Важно\!)

Чтобы атаки появлялись в логах, нужно включить галочку `Log` в правиле `Drop` вашего фаервола:

```mikrotik
/ip firewall filter set [find action=drop chain=input] log=yes log-prefix="FW_DROP"
```

### 4\. Убираем дубли из локального лога (Опционально)

Чтобы память роутера не забивалась атаками:

```mikrotik
/system logging set [find target=memory topics~"info"] topics=info,!firewall
```

-----

## Этап 4. Визуализация в Grafana

### Импорт панели Geomap

1.  В Grafana создайте новую панель (**New Dashboard** -\> **Add Visualization**).
2.  Выберите источник **Loki**.
3.  В настройках панели справа найдите иконку **"Panel JSON"** (или в меню Inspect -\> Panel JSON).
4.  Вставьте туда код ниже и нажмите **Apply**.

**JSON Панели:**

```json
{
  "type": "geomap",
  "title": "Карта атак (По странам)",
  "datasource": { "type": "loki", "uid": "ff5cy9c2r02kgb" },
  "targets": [
    {
      "refId": "A",
      "expr": "topk(200, sum by (geoip_country_name) (count_over_time({job=\"mikrotik_via_rsyslog\"} | unpack | geoip_country_name != \"\" [$__range])))",
      "queryType": "instant",
      "format": "table"
    }
  ],
  "transformations": [
    { "id": "labelsToFields", "options": { "mode": "columns" } },
    { "id": "merge", "options": {} },
    { "id": "convertFieldType", "options": { "fields": {}, "conversions": [ { "targetField": "Value", "destinationType": "number" } ] } },
    { "id": "organize", "options": { "renameByName": { "Value": "Количество атак", "geoip_country_name": "Страна", "Value #A": "Количество атак" } } }
  ],
  "fieldConfig": {
    "defaults": {
      "color": { "mode": "thresholds" },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "#00aeff", "value": 50 },
          { "color": "#EAB839", "value": 200 },
          { "color": "orange", "value": 500 },
          { "color": "red", "value": 1000 },
          { "color": "purple", "value": 5000 }
        ]
      }
    },
    "overrides": [
      { "matcher": { "id": "byName", "options": "Time" }, "properties": [ { "id": "custom.hideFrom", "value": { "legend": true, "tooltip": true, "viz": true } } ] },
      { "matcher": { "id": "byName", "options": "Количество атак" }, "properties": [ { "id": "color", "value": { "mode": "thresholds" } } ] }
    ]
  },
  "options": {
    "view": { "id": "coords", "lat": 55, "lon": 60, "zoom": 2, "allLayers": true },
    "layers": [
      {
        "type": "markers",
        "name": "Attacks",
        "config": {
          "style": {
            "size": { "fixed": 5, "min": 8, "max": 40, "field": "Количество атак" },
            "color": { "field": "Количество атак" },
            "opacity": 0.6,
            "symbol": { "mode": "fixed", "fixed": "img/icons/marker/circle.svg" }
          },
          "showLegend": true
        },
        "location": { "mode": "lookup", "lookup": "Страна" },
        "tooltip": true
      }
    ]
  }
}
```
<img width="803" height="402" alt="изображение" src="https://github.com/user-attachments/assets/8cb5af2b-4675-4c8d-ad4b-f8fe410add63" />


P.S.
Почему мы использовали Rsyslog?

Мы были вынуждены поставить Rsyslog между роутером и Promtail по трем техническим причинам:

**1. Нестандартная реализация Syslog в RouterOS v7**
В последних версиях RouterOS (v7) компания MikroTik изменила работу с логированием.
* Формат `remote-log-format=syslog` (который должен соответствовать стандарту **RFC 5424**) реализован с ошибкой: в заголовке пакета часто отсутствует цифра "версии протокола" перед датой.
* **Promtail** — это очень строгий парсер. Он ожидает идеального соблюдения стандарта. Когда он видел пакет от Микротика без версии, он выдавал ошибку: *`expecting a version value`* и отбрасывал пакет.
* **Rsyslog** — это старый, проверенный инструмент, который умеет "прощать" ошибки. Он принимает любые данные по UDP, не придираясь к заголовкам, и просто сохраняет их как текст.

**2. Проблема "фрейминга" (Framing Errors)**
Когда мы пытались переключить Микротик в режим `default` (без заголовков), Promtail начинал жаловаться на *`invalid or unsupported framing`*.
Promtail ожидает, что каждое сообщение будет иметь четкие границы (например, новую строку `\n` или длину сообщения), но по UDP пакеты приходят "как есть". Rsyslog умеет корректно собирать эти пакеты и складывать их в файл, разделяя строками, что идеально для Promtail.

**3. Проблема "Петли логирования" (Infinite Loop)**
Это была критическая проблема в конце.
* Loki (база данных) пишет свои служебные логи в системный журнал сервера (`/var/log/syslog`).
* Если бы мы настроили Promtail читать *весь* системный журнал, он бы отправлял в Loki логи самого Loki.
* Loki принимал бы их и снова писал в лог: "Я принял пакет".
* Это создало бы бесконечный цикл (шторм логов), который положил бы сервер.
* **Rsyslog** позволил нам отфильтровать трафик: *"Если данные пришли с IP 192.168.X.X (Микротик) — положи их в отдельный файл `mikrotik.log`"*. Это физически разделило потоки данных.

