
# Домашнее задание к занятию «Микросервисы: принципы»

Вы работаете в крупной компании, которая строит систему на основе микросервисной архитектуры.
Вам как DevOps-специалисту необходимо выдвинуть предложение по организации инфраструктуры для разработки и эксплуатации.

## Задача 1: API Gateway 

Предложите решение для обеспечения реализации API Gateway. Составьте сравнительную таблицу возможностей различных программных решений. На основе таблицы сделайте выбор решения.

Решение должно соответствовать следующим требованиям:
- маршрутизация запросов к нужному сервису на основе конфигурации,
- возможность проверки аутентификационной информации в запросах,
- обеспечение терминации HTTPS.
|   |   |   |   |
|---|---|---|---|
|   |   |   |   |
|   |   |   |   |  
  
| Решение | Маршрутизация на основе конфига | Аутентификация на основе запроса | HTTPS |
| --------- | --------------------------------- | ---------------------------------- | ------- |
| nginx   |   есть nginx.conf        |       есть           | поддерживает |
| kong    |    есть kong.yaml        |       есть           | поддерживает |
| Apache APISIX | Не обнаружил, настройка через curl  |  есть  | поддерживает |
| Tyk    | есть в json формате | Есть | поддерживает | 
  
На мой взгляд самым простым в реализации является nginx

Обоснуйте свой выбор.

## Задача 2: Брокер сообщений

Составьте таблицу возможностей различных брокеров сообщений. На основе таблицы сделайте обоснованный выбор решения.

Решение должно соответствовать следующим требованиям:
- поддержка кластеризации для обеспечения надёжности,  
Kafka, RabbitMQ, Redis
- хранение сообщений на диске в процессе доставки,  
Kafka
- высокая скорость работы,  
Kafka
- поддержка различных форматов сообщений,  
Kafka
- разделение прав доступа к различным потокам сообщений,  
Kafka, RabbitMQ
- простота эксплуатации.  
Kafka, RabbitMQ
  
Полагаю что для удовлетворения всех требований подходит kafka, т.к. у других решений осутствует часть функционала.
Обоснуйте свой выбор.

## Задача 3: API Gateway * (необязательная)

### Есть три сервиса:

**minio**
- хранит загруженные файлы в бакете images,
- S3 протокол,

**uploader**
- принимает файл, если картинка сжимает и загружает его в minio,
- POST /v1/upload,

**security**
- регистрация пользователя POST /v1/user,
- получение информации о пользователе GET /v1/user,
- логин пользователя POST /v1/token,
- проверка токена GET /v1/token/validation.

### Необходимо воспользоваться любым балансировщиком и сделать API Gateway:
Конфиг nginx со всеми редиректами:   
[nginx.conf]()  
  
**POST /v1/register**
1. Анонимный доступ.
2. Запрос направляется в сервис security POST /v1/user.

Ответ:  
Доработал скрипт server.py, т.к. в нем отсутствовал обрабочик /v1/user  
```python
@server.route('/v1/user', methods=['POST'])
def user():
    if not request.json or not 'login' in request.json or not 'password' in request.json:
        return make_response(jsonify({'error':'Bad request'})), 400
    
    login = request.json['login']
    password = request.json['password']

    data[login] = pbkdf2_sha256.hash(password)
    
    return login  
```
Командой делаем запрос к API, nginx(172.20.0.4) делает редирект в сервис security  
`curl -X POST -H 'Content-Type: application/json' -d '{"login":"petr", "password":"asdqwe"}' http://172.20.0.4/v1/user`  
Только я до сих пор так и не понял как он запоминает пользователей в dict. Я предполагал что dict хранится в рамках одного вызова server.py. Но даже после перезапуска контейнеров пользователи сохранились.

**POST /v1/token**
1. Анонимный доступ.
2. Запрос направляется в сервис security POST /v1/token.
Запрос:  
`curl -X POST -H 'Content-Type: application/json' -d '{"login":"bob", "password":"qwe123"}' http://172.20.0.4/v1/token`  

**GET /v1/user**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис security GET /v1/user.
Запрос:  
`curl -X GET -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I' http://172.20.0.4/v1/token/validation`
**POST /v1/upload**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис uploader POST /v1/upload.
Запрос:  
`curl -X POST -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I' -H 'Content-Type: octet/stream' --data-binary @1.PNG http://172.20.0.5:3000/v1/upload`

**GET /v1/user/{image}**
1. Проверка токена. Токен ожидается в заголовке Authorization. Токен проверяется через вызов сервиса security GET /v1/token/validation/.
2. Запрос направляется в сервис minio GET /images/{image}.
Этот обработчик тоже не нашел ни в server.js ни server.py. 
Не понятно как должна происходить авторизация при запросе файла.
### Ожидаемый результат

Результатом выполнения задачи должен быть docker compose файл, запустив который можно локально выполнить следующие команды с успешным результатом.
Предполагается, что для реализации API Gateway будет написан конфиг для NGinx или другого балансировщика нагрузки, который будет запущен как сервис через docker-compose и будет обеспечивать балансировку и проверку аутентификации входящих запросов.
Авторизация
curl -X POST -H 'Content-Type: application/json' -d '{"login":"bob", "password":"qwe123"}' http://localhost/token

**Загрузка файла**

curl -X POST -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJib2IifQ.hiMVLmssoTsy1MqbmIoviDeFPvo-nCd92d4UFiN2O2I' -H 'Content-Type: octet/stream' --data-binary @yourfilename.jpg http://localhost/upload

**Получение файла**
curl -X GET http://localhost/images/4e6df220-295e-4231-82bc-45e4b1484430.jpg

---

#### [Дополнительные материалы: как запускать, как тестировать, как проверить](https://github.com/netology-code/devkub-homeworks/tree/main/11-microservices-02-principles)

---

### Как оформить ДЗ?

Выполненное домашнее задание пришлите ссылкой на .md-файл в вашем репозитории.

---
