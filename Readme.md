# Исходники сайта [kuberops.ru](https://kuberops.ru)

<br/>

## Запустить локально с возможностью редактирования

Инсталлируете docker и docker-compose, далее:

```
$ cd ~
$ mkdir -p kuberops.ru && cd kuberops.ru
$ git clone --depth=1 https://github.com/webmakaka/kuberops.ru.git .
$ docker-compose up
```

<br/>

Остается в браузере подключиться к localhost:80

<br/>

## (Устарело!)

<br/>

### Запустить kuberops.ru на своем хосте с использованием docker контейнера:

```
$ docker run -i -t -p 80:80 --name kuberops.ru marley/kuberops.ru
```

<br/>

### Как сервис

```
$ sudo vi /etc/systemd/system/kuberops.ru.service
```

вставить содержимое файла kuberops.ru.service

```
$ sudo systemctl enable kuberops.ru.service
$ sudo systemctl start  kuberops.ru.service
$ sudo systemctl status kuberops.ru.service
```

http://localhost:4006

<br/><br/>

---

<br/>

**Marley**

Any questions in english: <a href="https://kuberops.ru/chat/">Telegram Chat</a>  
Любые вопросы на русском: <a href="https://kuberops.ru/chat/">Телеграм чат</a>
