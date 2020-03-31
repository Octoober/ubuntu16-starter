# Стартовая конфигурация Ubuntu 16.04 для Django проекта


Классика
```
apt-get update && apt-get upgrade
```
Изменить отступы tab для **VIM**. Просто создать файлик ```~/.vimrc``` и прописать в него:
```
set tabstop=4
set shiftwidth=4
set softtabstop=4
set expandtab
```
Так-то лучше!


## Установка Python 3.6 и дополнительня фигня
```
apt-get install python3.6 python3-pip python3-dev libpq-dev
pip3 install -U pip
python3.6 -V
```

### Установка **Virtualenv**
```
pip3 install virtualenv
```

***


# Django + Gunicorn + Nginx

Для примера, я буду использовать директорию ```/home/www/django/```
```
mkdir -p /home/www/django
cd /home/www/django
```

Окей. Теперь создаём виртуальное окружение, с именем ```venv```, например

```
virtualenv --python=/usr/bin/python3.6 venv
source venv/bin/activate
```

Теперь качаем **Django** и создаём проект ```firstsite```
```
pip install django
django-admin startproject firstsite
cd firstsite
```

В ```settings.py``` указываем ip или домен
```
vim firstsite/settings.py

---
ALLOWED_HOSTS = ['domain.com']
```

Попробуем запустить наш проект
```
python manage.py runserver 0.0.0.0:8000
```

А теперь открываем браузер, заходим на домен, который указали в ```ALLOWED_HOSTS``` с указанием порта :8000
```domain.com:8000```

Если всё гуд - едем дальше...

## Gunicorn

Теперь нужно поставить **Gunicorn** в наше **venv**
```
pip install gunicorn
```

Для теста запускаем проект через **Gunicorn**
```
cd /home/www/django/firstsite
gunicorn --bind 0.0.0.0:8000 firstsite.wsgi
```

Это нормально, что мы запускаем некий firstsite.wsgi, которого нет в нашем проекте, не паникуй.

### Сервисный файл Gunicorn

В ```systemd``` нужно создать файл ```gunicorn.service```
```
vim /etc/systemd/system/gunicorn.service

---
[Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=root
Group=www-data
WorkingDirectory=/home/www/django/firstsite
ExecStart=/home/www/django/venv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/home/www/django/firstsite/firstsite.sock firstsite.wsgi:application

[Install]
WantedBy=multi-user.target
```

Запускаем Gunicorn и добавляем в автозагрузку
```
systemctl start gunicorn
systemctl enable gunicorn
```

~~Шо там у хохлов~~ Что там в статусе gunicorn
```
systemctl status gunicorn
```
Надеюсь, что всё хорошо

Затем, проверяем наличие файла ```firstsite.sock``` в каталоге проекта
```
ls /home/www/django/firstsite
```

Если ```status``` выдал какую-то ошибку, проверяем файл ```gunicorn.service```, сравниваем пути

В моём случае, всё гуд. Если у вас так же, нужно перезапустить ~~Диму~~ демона, и перезапустить gunicorn
```
systemctl daemon-reload
systemctl restart gunicorn
```

## Nginx

Для начала, становим **Nginx**
```
apt-get install nginx
```

Идём настраивать
```
vim /etc/nginx/sites-available/firstsite

---
server {
    listen 80;
    server_name id_or_domain;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/www/django/firstsite/firstsite.sock;
    }
}
```

Создаём ссылку на наш конфиг в ```sites-enabled```
```
ln -s /etc/nginx/sites-available/firstsite /etc/nginx/sites-enabled
```

Проверяем nginx на ошибки, и перезапускаем, если всё хорошо
```
nginx -t
systemctl restart nginx
```


## Статичные файлы в Django

Поправим Nginx конфиг
```
vim /etc/nginx/sites-available/firstsite

---
server {
    ...
    location /static/ {
       root /home/www/django/firstsite
    }
    location /media/ {
        root /home/www/django/firstsite
    }
    ...
}
```
