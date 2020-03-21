# Server на Raspberry PI 3

## Установка статического IP

Заходим в файл конфигурации:

| $ | sudo nano /etc/dhcpcd.conf |
|---|-------------:|

В конце файла дописываем следующую строчку, чтобы игнорировать DHCP сервера, и назначить нужные нам настройки:
```
nodhcp
```
После этой строки назначим статический адрес для проводного подключения:
```
interface eth0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1

interface wlan0
static ip_address=192.168.1.100/24
static routers=192.168.1.1
static domain_name_servers=192.168.1.1
```

И перезагрузить:

| $ | sudo reboot |
|---|-------------:|

## Установка Django Apach2 wsgi:
Обновляем систему:

| $ | sudo apt-get update |
|---|:-------------|
| $ | sudo apt-get upgrade |

Устанавливает pip, Apach2, mod wsgi:

| $ | sudo apt-get install mc python3 python3-dev apache2 libapache2-mod-wsgi-py3 python3-pip |
|---|:-------------|
| $ | sudo pip3 install django |

Права:

| $ | sudo chmod -R 777 /home |
|---|:-------------|

Зайдем в папку:

| $ | cd /home |
|---|:-------------|

Создадим проект:

| $ | django-admin.py startproject Home |
|---|:-------------|

Зайдем в папку проекта:

| $ | cd /home/Home |
|---|:-------------|

Сделаем миграцию:

| $ | python3 manage.py makemigrations |
|---|:-------------|
| $ | python3 manage.py migrate |

Откроем файл:

| $ | sudo nano /etc/apache2/apache2.conf |
|---|:-------------|

И вставим в конце:
```python
ServerName localhost
```

Исправим корневую папку на с /var/www/html на /var/www

| $ | sudo nano /etc/apache2/sites-available/000-default.conf |
|---|:-------------|

Создать файл конфигурации Apach2 с именем проекта:

| $ | sudo nano /etc/apache2/sites-available/Home.conf |
|---|:-------------|

Вставить и изменить на свои пути:
```python
<VirtualHost *:80>
 ServerName 192.168.1.100
 DocumentRoot /home/Home
 WSGIScriptAlias / /home/Home/Home/wsgi.py

 # adjust the following line to match your Python path
 WSGIDaemonProcess mysite.example.com processes=2 threads=15 display-name=%{GROUP}
#python-home=/var/www/vhosts/mysite/venv/lib/python3.5
 WSGIProcessGroup mysite.example.com

 <directory /home/Home>
   AllowOverride all
   Require all granted
   Options FollowSymlinks
 </directory>

 Alias /static /home/Home/static

 <Directory /home/Home/static>
  Require all granted
 </Directory>
</VirtualHost>
```

Активируем наш файл:

| $ | sudo systemctl reload apache2 |
|---|:-------------|
| $ | sudo a2ensite Home |
| $ | sudo apachectl restart |

## Установка Samba

| $ | sudo apt-get install samba samba-common-bin |
|---|:-------------|

Теперь настроим Samba. Для этого открываем файл конфигурации:

| $ | sudo nano /etc/samba/smb.conf |
|---|:-------------|

И добавляем в конец файла следующие настройки:
```python
[Home]
Comment = Home folder
Path = /
Browseable = yes
Writeable = yes
only guest = no
create mask = 0777
directory mask = 0777
Public = yes
Guest ok = yes
```

Открываем доступ папкам:

| $ | cd /home |
|---|:-------------|
| $ | chmod 664 ./Home/db.sqlite3 |
| $ | chmod 775 ./Home |

Теперь надо дать группе www-data права:

| $ | sudo chown :www-data ./Home/db.sqlite3 |
|---|:-------------|
| $ | sudo chown :www-data ./Home |

Папкам:

| $ | sudo chmod -R 777 /home |
|---|:-------------|
| $ | sudo chmod -R 777 /var/log |




## Настроить WSGI в проекте:
В wsgi.py вставить:


```python
"""
WSGI config for DomoPhone project.

It exposes the WSGI callable as a module-level variable named ``application``.

For more information on this file, see
https://docs.djangoproject.com/en/2.1/howto/deployment/wsgi/
"""

import os
import time
import traceback
import signal
import sys

from django.core.wsgi import get_wsgi_application

sys.path.append('/home/DomoPhone')
# adjust the Python version in the line below as needed
#sys.path.append('/usr/lib/python3.5/site-packages')

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'DomoPhone.settings')

try:
    application = get_wsgi_application()
except Exception:
    # Error loading applications
    if 'mod_wsgi' in sys.modules:
        traceback.print_exc()
        os.kill(os.getpid(), signal.SIGINT)
        time.sleep(2.5)
```

В проекте Django в файле настроек settings.py вписать:
```python
ALLOWED_HOSTS = ["192.168.1.101", "127.0.0.1", "127.0.1.1"]
```
и там же внизу:

```python
STATIC_URL = '/static/'
STATIC_ROOT = 'static/'
```

## Собрать статические файлы
Создать в проекте папку static и:

| $ | cd /home/Home |
|---|:-------------|
| $ | sudo python3 manage.py collectstatic |

### Установка Celery

| $ | sudo pip3 install celery redis |
|---|:-------------|
| $ | sudo apt-get install redis-server |

Откройте файл настроек settings.py в вашем проекте Django
Добавим связанные с Celery/Redis конфиги:
```python
# celery
CELERY_BROKER_URL = 'redis://localhost:6379'
CELERY_RESULT_BACKEND = 'redis://localhost:6379'
CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TASK_SERIALIZER = 'json'
```

Потом нужно создать новый файл и добавить туда код
file: Home/Home/celery.py
```python
from __future__ import absolute_import, unicode_literals

import os

from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'Home.settings')

app = Celery('Home')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
app.autodiscover_tasks()


@app.task(bind=True)
def debug_task(self):
    print('Request: {0!r}'.format(self.request))
```

Добавить код в файл в той же дериктории:
```python
# proj/proj/__init__.py:
from __future__ import absolute_import, unicode_literals

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ('celery_app',)
```

Создать новый файл в своем приложении и добавить туда код tasks.py:
```python
# Create your tasks here
from __future__ import absolute_import, unicode_literals

from celery import shared_task


@shared_task
def add(x, y):
    return x + y


@shared_task
def mul(x, y):
    return x * y


@shared_task
def xsum(numbers):
    return sum(numbers)
```

Для проверки работы Celery:

| $ | cd /home/Home |
|---|:-------------|
| $ | celery -A Home worker -B -l INFO |

## Установка Mosquitto

| $ | sudo apt install mosquitto |
|---|:-------------|
| $ | sudo systemctl enable mosquitto.service |

Узнаем порт:

| $ | mosquitto -v |
|---|:-------------|

Узнаем IP:

| $ | hostname -I |
|---|:-------------|
