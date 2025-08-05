# Исходники сайта [docs.k8s.ru](https://docs.k8s.ru)

<br/>

## Запустить локально с возможностью редактирования

Инсталлируете docker и docker-compose, далее:

```
$ cd ~
$ mkdir -p docs.k8s.ru && cd docs.k8s.ru
$ git clone --depth=1 https://github.com/webmakaka/docs.k8s.ru.git .
$ docker-compose up
```

<br/>

Остается в браузере подключиться к localhost:80

<br/>

## (Устарело!)

<br/>

### Запустить docs.k8s.ru на своем хосте с использованием docker контейнера:

```
$ docker run -i -t -p 80:80 --name docs.k8s.ru marley/docs.k8s.ru
```

<br/>

### Как сервис

```
$ sudo vi /etc/systemd/system/docs.k8s.ru.service
```

вставить содержимое файла docs.k8s.ru.service

```
$ sudo systemctl enable docs.k8s.ru.service
$ sudo systemctl start  docs.k8s.ru.service
$ sudo systemctl status docs.k8s.ru.service
```

http://localhost:4006

<br/><br/>

---

<br/>

**Marley**

Any questions in english: <a href="https://docs.k8s.ru/chat/">Telegram Chat</a>  
Любые вопросы на русском: <a href="https://docs.k8s.ru/chat/">Телеграм чат</a>
