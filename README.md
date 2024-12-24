# **Связка NextCloud + Onlioffice в локальной сети**

1) Для начала клонируем репозитория 

> **git clone**

2) Перейдем в папку

> **cd nextcloud**

3) Создадим файл необходимыми переменными

> **nano .env**

Вставим необходимые данные (пример):

```envtext
JWT_ENABLED=true
JWT_SECRET=<ВАШ СЕКРЕТНЫЙ КЛЮЧ>
JWT_HEADER=Authorization
JWT_IN_BODY=true
POSTGRES_DB=nextcloud
POSTGRES_USER=nextcloud
POSTGRES_PASSWORD=pass
POSTGRES_HOST=postgres-nextcloud
NEXTCLOUD_ADMIN_USER=user
NEXTCLOUD_ADMIN_PASSWORD=12345678
NEXTCLOUD_TABLE_PREFIX=oc_
NEXTCLOUD_TRUSTED_DOMAINS=cloud.home.loc
```

4) Запустим Docker-compose на выполнение 

> **docker-compose up -d**

5) Переходим к установке nginx:

- Для начала нужно обновить списки пакетов из репозиториев:

> **sudo apt update**

- После окончания процесса обновления пакетов можно установить Nginx на машину:

> **sudo apt install nginx**

- Дождемся окончания установки, а после добавим программу в автозагрузку:

> **sudo systemctl enable nginx**

- Теперь нужно проверить, что веб-сервер успешно установлен и работает, а также добавлен в автозагрузку. Проверим статус работы веб-сервера:

> **sudo service nginx status**

- Теперь проверим его наличие в автозагрузке:

> **sudo systemctl is-enabled nginx**

- После создадим конфигурационный файл сайта в папке `sites-available`:

> **sudo nano /etc/nginx/sites-available/cloud.home.loc**

и вставим в него следующий текст (надо разобраться с прокси)

```javascript
proxy_set_header Upgrade $http_upgrade;
#proxy_set_header Connection $proxy_connection;
proxy_set_header X-Forwarded-Host $http_host/editors;
server {
        listen 80;
        server_name cloud.51kaf.loc;
        location / {
            proxy_pass_header Server;
            proxy_pass http://<ВАШ IP>:8081/;
        }
        location /editors/ {
            proxy_pass http://<ВАШ IP>:8082/; 
        }
}
```

- Последнее, что осталось сделать, — это создать ссылку в директории `sites-enabled` на конфигурацию сайта cloud.home.loc, чтобы добавить его из доступных во включенные:

> **sudo ln -s /etc/nginx/sites-available/cloud.home.loc /etc/nginx/sites-enabled/**

- После создания виртуального хоста проведем тестирование конфигурации:

> **sudo nginx -t**

- Отключим сайт по умолчанию, удалив запись о дефолтном виртуальном хосте:

>  **sudo rm /etc/nginx/sites-enabled/default**

- Перезагружаем веб-сервер:

> **sudo systemctl restart nginx**

6) Редактируем файл конфигурации для связи onlyoffice c Nextcloud

> **sudo nano config/config.php**

И собственно вставляем в конец кода

```jsontext
<?php
$CONFIG 
......

  'onlyoffice' => array (
    'verify_peer_off' => true,
    'jwt_secret' => '<ВАШ СЕКРЕТНЫЙ КЛЮЧ>',
    'jwt_header' => 'Authorization',
)
......

);
```

7) Устанавливаем приложение onlyoffice в nextcloud,  прописываем адрес ***<ВАШ IP>:8082,*** (возможно если прописать в hosts, заработает символьная ссылка)

# **UFW — начальная настройка в Ubuntu**

Произведём начальную настройку UFW на внешнем сервере Ubuntu Server 22.04.4. Работаем под **sudo**.

Текущая версия UFW:

> *ufw --version*

Текущий статус UFW по умолчанию inactive:

> *ufw status*

Перед тем как приступить к настройке UFW, нужно позаботиться о том, чтобы не наступить на грабли. А именно: не закрыть себе доступ. Если у вас есть доступ к консоли сервера, уже хорошо, вы сможете что-то исправить. А если вы работаете через OpenSSH по 22 порту, то можно случайно себя заблокировать.

> sudo ufw allow OpenSSH

или

> sudo ufw allow 22

Я работаю из сети, у которой есть прямой IP адрес, добавлю его в разрешающее правило:

> *ufw allow from 172.28.28.99*

Теперь с указанного IP можно получить доступ ко всем портам сервера, включая TCP 22, через который у меня открыта SSH сессия.

Включаем UFW:

> *ufw enable*

Проверим статус:

> *ufw status*

Настроим правила по умолчанию.

- Все входящие соединения блокируем.
- Все исходящие соединения разрешаем

> *ufw default deny incoming  
> ufw default allow outgoing*

### **Добавим правила UFW**

Мы защитились со всех сторон, иногда этого хватает, если сервер не требует внешних соединений. Пусть у нас сервер будет выполнять роль web-сервера, разрешим входящие соединения на TCP 80 (HTTP) и TCP 443 (HTTPS).

> *ufw allow 80/tcp  
> ufw allow 443/tcp  
> ufw status*

**Отключить UFW**. Для этого используется команда

> sudo ufw disable

Если что-то пошло не так, можно использовать команду reset для сброса настроек до состояния по умолчанию:

> sudo ufw reset
