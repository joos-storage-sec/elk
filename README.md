# elk
ELK

Стек ELK — это аббревиатура, используемая для описания стека, состоящего из трёх популярных проектов:
- Elasticsearch
- Logstash
- Kibana
Стек ELK предоставляет возможность:
- агрегировать журналы из всех ваших систем и приложений
- анализировать эти журналы
- создавать визуализации для мониторинга приложений и инфраструктуры, более быстрого устранения неполадок, анализа безопасности и многого другого
Есть также платный Elastic Cloud и коммерческая версия ELK-стека

**Beats** — это отправители данных с открытым исходным кодом, которые вы устанавливаете в качестве агентов на своих серверах для отправки данных в Elasticsearch. На данный момент есть Auditbeat, Filebeat, Functionbeat, Heartbeat, Journalbeat, Metrics, Packetbeat, Winlogbeat. Мы посмотрим только на Filebeat

**Elasticsearch** — это распределённая, поисковая и аналитическая система, которая является сердцем ELK-стека. Он централизованно хранит данные для поиска, точной настройки релевантности и мощной аналитики, легко масштабируется. Все данные, которые будут писаться системой поставки, будут оседать и индексироваться в Elasticsearch

На основе Elasticsearch строят не только системы поставки логов, но и сервисы для поиска бизнесовых данных для пользователей, например, ebay classifieds. Данные в виде документов поставляются через API или тулзы вроде Logstash или Beats. После записи в базу поверх данных автоматически строятся индексы для быстрого поиска по полям через API или Kibana

**Установим Elasticsearch на Debian 10:**
```
# apt update && apt install gnupg apt-transport-https <--зависимости
# wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key
add - <--добавляем gpg-ключ
# echo "deb [trusted=yes] https://mirror.yandex.ru/mirrors/elastic/7/ stable
main" | sudo tee /etc/apt/sources.list.d/elastic-7.x.list <--добавляем
репозиторий в apt
# apt update && apt-get install elasticsearch <--устанавливаем elastic
# systemctl daemon-reload <--обновляем конфиги systemd
# systemctl enable elasticsearch.service <--включаем юнит
# systemctl start elasticsearch.service <--запускаем сервис
```
После установки базы первым делом обезопасьте её. И настройте бекапы

Проверяем, что сервер запустился:
```
# curl 'localhost:9200/_cluster/health?pretty'
{
"cluster_name" : "netology-logging",
"status" : "green",
"timed_out" : false,
"number_of_nodes" : 1,
"number_of_data_nodes" : 1,
"active_primary_shards" : 1,
"active_shards" : 1,
"relocating_shards" : 0,
"initializing_shards" : 0,
"unassigned_shards" : 0,
"delayed_unassigned_shards" : 0,
"number_of_pending_tasks" : 0,
"number_of_in_flight_fetch" : 0,
"task_max_waiting_in_queue_millis" : 0,
"active_shards_percent_as_number" : 100.0
}
```
Немного настройки в /etc/elasticsearch/elasticsearch.yml:
```
cluster.name: netology-logging <--меняем имя кластера
node.name: node-1 <--меняем название ноды, если нужно
node.roles: [ master, data, ingest ] <--какую функцию будет выполнять эта нода
cluster.initial_master_nodes: ["node-1"] <--узлы, участвующие в голосовании по
выбору мастера
discovery.seed_hosts: ["ip-адрес"] <--список возможных мастеров кластера
path.data: /var/lib/elasticsearch <-где храним данные
path.logs: /var/log/elasticsearch <--куда пишем логи
network.host: 0.0.0.0 <--какой ip слушает хост
# systemctl restart elasticsearch
```
**Kibana** — это бесплатный и открытый пользовательский интерфейс, который позволяет визуализировать данные Elasticsearch. Интерфейс отчасти похож на Grafana

Возможности Kibana:
- визуализация данных
- аналитика
- мониторинг и алертинг
- ML

**Установим Kibana на Debian 10:**
```
# apt install kibana <--установка
# systemctl daemon-reload <--обновляем конфиги systemd
# systemctl enable kibana.service <--включаем юнит
# systemctl start kibana.service <--запускаем сервис
```
Настройки в /etc/kibana/kibana.yml:
```
server.host: "0.0.0.0" <--открываем интерфейс в мир
# systemctl restart kibana
```
**Logstash** — это сервис сбора данных с открытым исходным кодом и с возможностями конвейерной обработки в реальном времени. Logstash может динамически объединять данные из разрозненных источников и нормализовывать данные в места назначения по вашему выбору. Брать любые данные, парсить, нормализовывать их и писать в Elasticsearch или в любой поддерживаемый провайдер. Хранит стейт в файле

Конфигурация Logstash делится на:
- inputs. Отвечает за то, откуда Logstash возьмёт данные, например, из файла, syslog, stdin или redis
- filters. Как logstash изменит данные, которые пришли из inputs. Какие поля удалит, какие поменяет
- outputs. Куда после преобразования данные будут отправлены: в elasticsearch или file, например
- codecs. Сериализация. Например, преобразование строки в json или наоборот
**Установим Logstash на Debian 10:**
```
# apt install logstash <--установка
# systemctl daemon-reload <--обновляем конфиги systemd
# systemctl enable logstash.service <--включаем юнит
# systemctl start logstash.service <--запускаем сервис
```
Настроим поставку access-лога nginx в elasticsearch:
```
input {
  file {
    path => "/var/log/nginx/access.log"
    start_position => "beginning"
  }
}
filter {
  grok {
    match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user_name} \[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:agent}\"" }
  }
  mutate {
    remove_field => [ "host" ]
  }
}
  output {
    elasticsearch {
      hosts => "178.154.215.248"
      data_stream => "true"
    }
}
```
**Filebeat** — это легковесный агент для пересылки и централизации данных из файлов. Устанавливается как демон на сервера. Основное отличие от Logstash — лёгкость и скорость, но с урезанным функционалом пайплайнов. Так же, как Logstash, хранит стейт в файле и может менять данные перед отправкой

Конфигурация состоит из двух компонентов:
- inputs. Как и откуда будут читаться данные для поставки
- processors. Позволяет незначительно менять данные в пайплайне
- harvester (комбайн). Запускается на каждый файл, который читает Filebeat, собирательное название каждой поставки данных

**Установим Filebeat на Debian 10:**
```
# apt install filebeat <--установка
# systemctl daemon-reload <--обновляем конфиги systemd
# systemctl enable filebeat.service <--включаем юнит
# systemctl start filebeat.service <--запускаем сервис
```
конфиг для отправки в Logstash:
```
# меняем конфиг Logstash
input {
  beats {
    port => 5044
  }
}
# меняем конфиг Filebeat
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/nginx/access.log
processors:
  - drop_fields: <--удаляются системные поля, которые добавил filebeat
    fields: ["beat", "input_type", "prospector", "input", "host", "agent", "ecs"]

output.logstash:
  hosts: ["178.154.215.248:5044"]
```
