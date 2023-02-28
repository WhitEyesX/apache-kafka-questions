# apache-kafka-questions

# Kafka Producer
+ [Базовая конфигурация Kafka Producer](#Базовая-конфигурация-kafka-producer)
+ [Producer Acknowledgement](#Producer-acknowledgement-(acks))
+ [Acks = all & min.insync.replicas](#acks--all--mininsyncreplicas)
+ [Kafka Topic Availability (Рассматриваем replication factor = 3)](#kafka-topic-availability-рассматриваем-replication-factor--3)
+ [Producer Retries](#Producer-retries)
+ [Producer Timeouts](#Producer-timeouts)
+ [Idempotent Producer](#Idempotent-producer)

## Базовая конфигурация Kafka Producer
+ bootstrap server
+ key serializer
+ value serializer

## Producer Acknowledgement (acks)
+ ```ack = 0```: Producer не ждет подтверждения получения сообщения от Kafka
+ ```ack = 1```: Producer ждет подтверждения только от broker-leader
+ ```ack = -1(all)```: Producer ждет подтверждения от всех partition (Leader + Replicas)

## Acks = all & min.insync.replicas
При ```acks = all```, broker-leader проверяет, достаточно ли доступных реплик для безопасной записи данных(контролируется параметром min.insync.replicas)
+ ```min.insync.replicas = 1```: Только broker-leader должен отправить ack
+ ```min.insync.replicas = 2```: По крайней мере broker-leader
Если параметер ```min.insync.replicas = 2``` и мы отправляем данные в Kafka, то в случае отсутсвия двух работоспособных brokers Kafka вернет ошибку ```NOT_ENOUGH_REPLICAS```

## Kafka Topic Availability (Рассматриваем replication factor = 3)
+ ```acks = 0``` & ```acks = 1```: если есть хотябы один работоспособный брокер, то topic доступен для записи.
+ ```acks = all```
  + ```min.insync.replicas = 1 (default)```: если есть хотябы один работоспособный брокер, то мы можем допустить отказ двух брокеров
  + ```min.insync.replicas = 2```: topic должен иметь 2 работоспособных ISR, таким образом мы можем допустить отказ максимум одного брокера
  + ```min.insync.replicas = 3```: не имеет никакого смысла, так как мы не сможем предоставить доступность topic, при сбое в любом брокере
  
В общем, при ```ack = all``` & ```replication factor = N``` & ```min.insync.replicas = M```, мы можем перетерпеть N - M упадков брокеров
+ ```ack = all``` & ```min.insync.replicas = 2``` (replication factor = 3) - наиболее популярная настройка для надеждности/доступности данных, позволяющая допустить отказ единственного брокера

## Producer Retries
В случае кратковременных отказов ожидается, что разработчики обработают исключения иначе данные будут утеряны (как пример NOT_ENOUGH_REPLICAS)
Существует настройка ```retries```
+ ```default: 0 (Kafka <= 2.0)```
+ ```default: Integer.MAX_VALUE(2^31 - 1) (Kafka >= 2.1)```
Также ```retries.backoff.ms``` (default - 100ms) которая отвечает за задержку до следующего retry

## Producer Timeouts
+ Если ```retries > 0```, общее врекмя retries ограничены временем.
+ Kafka >= 2.1, можно выставить ```delivery.timeout.ms = 120000(2 min)``` таким образом данные будут утеряны если не будет получено acknowledgement в течении delivery.timeout.ms
  ### Producer retries: WARNING for old Kafka versions
  + Если не используется idempotent produceer (не рекомендуется для старых версий)
    + В случае retries есть шанс, что данные будут отправлены в неправильном порядке
    + Если вы полагаетесь на key-based ordering это может стать проблемой
 
  Для решения этой проблемы нужно установить настройку ```max.in.flight.requests.per.connection``` - максимальное количество unacknowledged запросов могут быть отправлены в параллели.
  
## Idempotent Producer
Producer может отправлять дупликаты в Kafka из за сетевых ошибок
В Kafka > 0.11, можно выставить настройку ```enable.idempotence = true```. Если producer делает дупликат данных, то Kafka не запишет его снова, а просто отправит acknowledgement. Делается это при помощи присваивания каждому запроса некого ```sequence number``` (вкратце).
+ Это гарантирует стабильный и надежный pipeline
+ Это настройка включена by default для Kafka >= 3.0

Эта настройка также устанавливает(если они не заданы вручную) такие параметры как:
+ ```retries = Integer.MAX_VALUE```
+ ```max.in.flight.requests.per.connection = 1``` (Kafka 0.11)
+ ```max.in.flight.requests.per.connection = 5``` (Kafka >= 1.0) - higher performance and keep ordering [KAFKA-5494]
+ ```acks = all```
```
Since Kafka 3.0 Kafka Producer is SAFE by default. (acks = all & enable.idempotence = true)
With Kafka 2.8 and lower the Kafka Producer comes with acks = 1 and enable.idempotence = false, 
its recommended to use SAFE Producer and change these settings
```
