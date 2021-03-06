---
tags: Advanced Networking
---
:::success
# AN Project
Name: Ivan Okhotnikov
:::

## Socat

:::info
Socat - программа, которая позволяет переадресовывать сокеты с компьютера хоста клиенту

### Переадресация docker.sock

План:
- Переименуем оригинальный сокет
- Переадресуем подключения TCP в запросы к сокету
- Создаем новый сокет (для использования основной машиной)


Для задачи переадресации нам поможет команда:

```
sudo socat TCP-LISTEN:8089,reuseaddr,fork UNIX-CONNECT:/var/run/docker.sock.origin 
```
В ней мы указываем что соединяться необходимо с сокетом докера и разрешаем возможность соединения через TCP (порт 8089)

Далее с помощью команды:
```
sudo socat UNIX-LISTEN:/var/run/docker.sock,fork TCP-CONNECT:127.0.0.1:8089
```
Мы создаем сокет для использования основной машиной. Все запросы к этому сокету будут переадресованы в TCP, а он в свою очередь обратится к оригинальному сокету (схема представлена ниже)

![](https://i.imgur.com/VFKYhAB.png)
:::


## Управление доступом к сокету на хосте докера 
:::info

Поскольку подключение к сокету напрямую на удаленной машине невозможно, существует два подхода:
- Раздача с помощью tcp и далее либо подключение по tcp, либо интерперация в socket на удаленной машине
- Подключение с помощью ssh

## Tcp socket sharing (without TLS)
Как говорилось ранее, можно с помощью socat подключиться к удаленному хосту:
```
sudo socat UNIX-LISTEN:/var/run/docker.sock,fork TCP-CONNECT:172.17.0.1:8089
```
И тем самым подменить сокет для использования.

Главным недостатком данного подхода является остутствие шифрования данных
Это дает возможность злоумышленникам манипулировать работающими контейнерами (смотреть логи, выполнять команды) и создавать новые с целью dos атаки на машину хоста

## Tcp socket sharing (with TLS)
Данный подход позволяет производить коммуникацию с docker с помощью TLS, что позволяет быть уверенным в безопасности подключений и взаимодействий

![](https://i.imgur.com/ehxK6nf.png)


### Создание сертификата сервера
Для данного подхода необходимо будет создать ключ и TLS CA:
Корневой ключ для большей безопасности защитим c помощью aes256 ключа
```
openssl genrsa -aes256  -out ca-key.pem 4096
openssl req -new -x509 -days 365 -key ca-key.pem -sha256 -out ca.pem
```
Важно при создании сертификата указать в качестве `Common Name` IP адресом или DNS именем!
![](https://i.imgur.com/EJIAq65.png)


После создания корневого сертификата, необходимо создать сертификат сервера (переменную $HOST надо заменить), в котором мы укажем хост или доменные имя, которое указывали на предыдущем шаге при создании корневого сертификата
```
openssl genrsa -out server-key.pem 4096
openssl req -subj "/CN=st11" -sha256 -new -key server-key.pem -out server.csr
```

Теперь создадим конфигурационный файл, в котором укажем адреса, по которым возможно tls подключения и установим атрибуты  расширенного использования ключа демона докер, использующиеся для аутентификации на сервере:

`extfile.cnf`
```
subjectAltName = DNS:st11,IP:172.17.0.1,IP:127.0.0.1
extendedKeyUsage = serverAuth
```

Осталось только сгенерировать подписанный сертификат:
```
openssl x509 -req -days 365 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem -extfile extfile.cnf
```
![](https://i.imgur.com/s7SNEC1.png)


### Создание сертификата клиента

Создам сертификат и ключ для клиентского сертификата:
```
openssl genrsa -out key.pem 4096
openssl req -subj '/CN=client' -new -key key.pem -out client.csr
```

Также создам новый конфигурационный файл для подписи, указывая что данный сертификат будет использоваться для подписи
```
echo extendedKeyUsage = clientAuth > extfile-client.cnf
```

Подписываю сертификат с помощью корневого сертификата и его ключа
```
openssl x509 -req -days 365 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile-client.cnf
```

Чтобы демон принимал соединения только от доверенных клиентов, необходимо его запустить используя сертификаты следующим образом:
```
dockerd \
    --tlsverify \
    --tlscacert=../ca.pem \
    --tlscert=../server-cert.pem \
    --tlskey=../server-key.pem \
    -H=0.0.0.0:2376
```
Также я разрешаю всем клиентам подключаться с помощью 2376 порта (он стандартный для докера и клиентам его можно не указывать)

## SSH socket sharing
Данный подход обладает большей безопасностью от несанкционированных подключений, поскольку все взаимодействие с сокетом происходит внутри ssh трафика

![](https://i.imgur.com/HHULVnu.png)


Для настройки необходимо, чтобы на хост с докер сокетом был установлен ssh-server

Предварительные работы:
1 - для корректной работы в файле конфигурации ssh подключений `~/.ssh/config` у клиента должно быть следующие настройки:

```
ControlMaster     auto
ControlPath       ~/.ssh/control-%C
ControlPersist    yes
```
2 - Поскольку ввод пароля в данном подходе не предполагается, подключение происходит исключительно посредством ssh ключей. Публичный ключ клиента ОБЯЗАТЕЛЬНО должен храниться у ssh сервера

Иначе же при попытке подключения будет получена следующая ошибка:
![](https://i.imgur.com/bfENoxt.png)


Сгенерировать ключ на клиенте можно с помощью команды: `ssh-keygen -o`

Теперь можно указывать ssh подключение в контексте докера или переменной окружения на клиентской машине:

```
export DOCKER_HOST=ssh://st11@172.17.0.1
```
:::

## Варианты подключения к докер-сокету на удаленной машине

:::info
Есть еще пара подходов для этого:

### Указание в переменной окружения для обычного tcp соединения
Данный подход позволяет очень просто настроить вариант подключения, но в нем и есть недостатки:
- После перезагрузки приходится заново его настраивать
- Нет возможности быстрого переключения между несколькими окружениями
```
export DOCKER_HOST=tcp://172.17.0.1
```

### Указание в переменной окружения для tcp соединения с использованием TLS
Данных подход похож на предыдущий, но для него нужно немного доработать, указав в переменной `DOCKER_TLS_VERIFY` флаг `1`

```
export DOCKER_HOST=tcp://172.17.0.1:2376 DOCKER_TLS_VERIFY=1
```

Также необходимо переместить сертификаты безопасности в директорию `~/.docker`

```
mkdir -pv ~/.docker
cp -v {ca,cert,key}.pem ~/.docker
```

### Добавление контекста
Данный подход позволяет сохранить несколько вариантов подключений и далее уже использовать их на свой выбор

Добавить новый контекст очень просто - достаточно лишь ввести команду:

```
docker context create \
    --docker host=tcp://172.17.0.1:8089 \
    --description="Remote engine" \
    my-remote-engine
```
Необходимо задать контексту название и хост, с помощью которого будет происходить управление движком докера
В данной команде можно добавить описание для удобства.

Для просмотра всех контекстов можно воспользоваться командой:
```
docker context ls
```
![](https://i.imgur.com/eNOWWD9.png)
Здесь звездочка рядом с именем показывает текущий контекст


Для выбора необходимого контекста можно воспользоваться командой:
```
docker context use my-remote-engine
```
В ней нужно указать имя необходимого контекста
![](https://i.imgur.com/CgJ2v4Y.png)

Удалить контекст можно соответственно (но перед этом обязательно нужно переключиться на другой контекст):
```
docker context rm my-remote-engine
```
:::

## Перехват трафика

Поскольку ранее мы включили переадресацию с tcp в unix socket, становится доступен механизм перехвата трафика

Это можно сделать с помощью tcpdump:

```
sudo tcpdump -i lo -netvv port 8089 -w 1.cap
```
Здесь мы указываем loopback интерфейс, порт и файл вывода захваченного трафика
![](https://i.imgur.com/bvzEb2B.png)



## Отправка запросов к сокету

:::info

Отправить запрос к сокету не составляет особого труда, например сама программа `docker <command>` работает именно по такому принципу


Для отправки запроса мы можем воспользоваться командой `Curl`, которая предоставляет такую возможность с помощью флага  `--unix-socket` с указанием пути расположения сокета.

Для примера я получу список докер-образов на машине в формате JSON, а с помощью программы `JQ` я сделаю вывод в консоли более читаемым
```
curl --unix-socket /var/run/docker.sock http://localhost/images/json | jq
```

![](https://i.imgur.com/3Ms08bM.png)


Также поскольку ранее мы открыли tcp-порт, мы можем обратиться через tcp

```
curl 127.0.0.1:8089/images/json | jq
```

![](https://i.imgur.com/Ec1iImn.png)
:::

## Возврат к захваченному трафику

:::info
Программа tcpdump уже успела записать трафик, который мы отправляли к сокету и пора его посмотреть

![](https://i.imgur.com/5Fk2B83.png)

Для просмотра захваченных пакетов можно воспользоваться программой wireshark

```
wireshark 1.cap
```

Посе запуска команда запустит программу wireshark и в его окне появятся все перехваченные пакеты

![](https://i.imgur.com/1bIIzeM.png)

Хорошо видно что трафик не шифруется, поскольку и запрос и ответ от сервера находятся в чистом виде

![](https://i.imgur.com/jqVGKlq.png)

### Трафик от программы docker
Посмотрев захваченных трафик от команды `docker ps`, хорошо стадии запросов:

1. Проверка ответа от сокета - `_ping`
2. Сам запрос
![](https://i.imgur.com/E87VRe6.png)


Что касается ответа от сокета - все также ничего не зашифровано и ответ приходит в чистом виде
![](https://i.imgur.com/EAJjPM9.png)

:::