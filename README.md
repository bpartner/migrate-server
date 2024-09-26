# Инструкция по переносу веб-платформы Syntaxis.

## 1. Подготовка нового сервера

Перед началом переноса убедитесь, что новый сервер соответствует всем требованиям:

- Операционная система: Рекомендуется использовать ту же ОС, что и на старом сервере, для упрощения процесса.
- Установите необходимые пакеты и зависимости:

  ```bash
   #Обновите список пакетов
  sudo apt update && sudo apt upgrade -y

   #Установите PHP 8.2 и необходимые расширения
  sudo apt install -y php8.2 php8.2-mbstring php8.2-xml php8.2-mysql php8.2-zip php8.2-curl

   #Установите MySQL 5.7
  sudo apt install -y mysql-server=5.7.*

   #Установите необходимые сервисы
  sudo apt install -y redis-server supervisor git curl unzip

   #Установите Composer
  curl -sS https://getcomposer.org/installer | php
  sudo mv composer.phar /usr/local/bin/composer

  # Установите веб-сервер (например, Nginx)
  sudo apt install -y nginx

   #Установите и настройте почтовые сервисы (Postfix для SMTP, Dovecot для POP3/IMAP)
   #Если будет использоваться внешний почтовый сервис, этот шаг можно пропустить
  sudo apt install -y postfix dovecot-imapd dovecot-pop3d
  ```

## 2. Настройка веб-сервера (Nginx)

Создайте конфигурационный файл для вашего сайта:

```bash
sudo nano /etc/nginx/sites-available/your_domain.conf
```

Пример конфигурации для Laravel:

```nginx
server {
    listen 80;
    server_name your_domain.com www.your_domain.com;

    root /var/www/your_project/public;
    index index.php index.html index.htm;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";

    charset utf-8;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
```

Активируйте сайт и перезапустите Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/your_domain.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

## 3. Клонирование репозитория или перенос файлов

### Через Git:

1. Установите Git на новом сервере, если еще не установлен:

    ```bash
    sudo apt install -y git
    ```

2. Клонируйте репозиторий в нужную директорию:

    ```bash
    cd /var/www/
    git clone https://github.com/yourusername/yourrepository.git your_project
    composer install
    
    ```
3. Перейдите в директорию проекта:  
    ```bash
      cd /var/www/your_project
    ```
4. Установите зависимости Composer:  
    ```bash
    composer install
    ```
5. Скопируйте файл .env с конфигурациями из старого сервера:  
    ```bash
    scp user@old_server_ip:/path/to/your_project/.env /var/www/your_project/
    ```
6. Обновите параметры .env на новом сервере, такие как подключения к базе данных, пути и т.д.  
7. Сгенерируйте ключ приложения:  
    ```bash
    php artisan key:generate
    ```
8. Выполните миграции и заполнение базы данных, если необходимо:  
    ```bash
    php artisan migrate --force
    php artisan db:seed --force
    ```
9. Настройте права доступа:
   ```bash
   sudo chown -R www-data:www-data /var/www/your_project
   sudo chmod -R 755 /var/www/your_project
   sudo chmod -R 775 /var/www/your_project/storage
   sudo chmod -R 775 /var/www/your_project/bootstrap/cache
    ```
### Через SCP/SFTP:

1. Архивируйте проект на старом сервере:

    ```bash
    cd /path/to/your_project
    tar -czvf your_project.tar.gz .
    ```

2. Перенесите архив на новый сервер:

    ```bash
    scp your_project.tar.gz user@new_server_ip:/var/www/your_project/
    ```

3. Распакуйте архив на новом сервере:

    ```bash
    cd /var/www/your_project
    tar -xzvf your_project.tar.gz
    ```

### Настройка окружения

   1. Скопируйте файл `.env` с конфигурациями из старого сервера:
       ```bash
       scp user@old_server_ip:/path/to/your_project/.env /var/www/your_project/
       ```
   2. Обновите параметры `.env` на новом сервере, такие как подключения к базе данных, пути и т.д.
        - удалите раздел #dev only
      
### Перенос базы данных
    
   1. Экспортируйте базу данных на старом сервере:

      ```bash
      mysqldump -u your_db_user -p your_database > your_database.sql
      ```

   2. Перенесите дамп на новый сервер:

       ```bash
       scp your_database.sql user@new_server_ip:/home/user/
       ```

   3. Импортируйте базу данных на новом сервере:

       ```bash
       mysql -u your_db_user -p your_database < /home/user/your_database.sql
       ```
### Установка зависимостей и оптимизация

1. Перейдите в директорию проекта:

    ```bash
    cd /var/www/your_project
    ```

2. Установите зависимости Composer:

    ```bash
    composer install --optimize-autoloader --no-dev
    ```

3. Сгенерируйте ключ приложения:

    ```bash
    php artisan key:generate
    ```

4. Выполните миграции и заполнение базы данных, если необходимо:

    ```bash
    php artisan migrate --force
    php artisan db:seed --force
    ```
5. Создайте символическую ссылку для хранилища:

    ```bash
    php artisan storage:link
    ```
   
6. Оптимизируйте проект:

    ```bash
    php artisan optimize
    ```

7. Настройте права доступа:

     ```bash
     sudo chown -R www-data:www-data /var/www/your_project
     sudo chmod -R 755 /var/www/your_project
     sudo chmod -R 775 /var/www/your_project/storage
     sudo chmod -R 775 /var/www/your_project/bootstrap/cache
     ```

### Настройка Redis и Supervisor

1. Настройте Redis:

   Убедитесь, что Redis работает и настроен в вашем `.env` файле:

    ```env
    REDIS_HOST=127.0.0.1
    REDIS_PASSWORD=null
    REDIS_PORT=6379
    ```

   Запустите и активируйте Redis:

    ```bash
    sudo systemctl start redis
    sudo systemctl enable redis
    ```

2. Настройте Supervisor для управления очередями Laravel:

   Установите Supervisor, если он еще не установлен:

    ```bash
    sudo apt install -y supervisor
    ```

   Создайте конфигурационный файл для вашего приложения:

    ```bash
    sudo nano /etc/supervisor/conf.d/laravel-worker.conf
    ```

   Пример конфигурации:

     ```
        [program:horizon]
        process_name=%(program_name)s
        command=php /path/to/app/artisan horizon
        autostart=true
        autorestart=true
        user=root
        redirect_stderr=true
        stdout_logfile=/path/to/log/horizon.log
        stopwaitsecs=3600
     ```

    Примените изменения:
     ```bash
         sudo supervisorctl reread
         sudo supervisorctl update
         sudo supervisorctl start laravel-worker:*
     ```

### Настройка Cron задач

1. Откройте crontab для пользователя `www-data` или нужного пользователя:

    ```bash
    sudo crontab -u www-data -e
    ```

2. Добавьте задание для запуска Laravel Scheduler каждую минуту:

    ```cron
    * * * * * php /var/www/your_project/artisan schedule:run >> /dev/null 2>&1
    ```

   Сохраните и выйдите из редактора.

### Настройка почтовых сервисов (POP3, IMAP, SMTP с SSL)

1. Настройте Postfix для SMTP:

   При установке Postfix выберите тип конфигурации «Internet Site» и укажите ваше доменное имя.

2. Настройте Dovecot для POP3 и IMAP:

   Основные настройки обычно находятся в `/etc/dovecot/dovecot.conf`. Убедитесь, что протоколы POP3 и IMAP включены.

### Настройте SSL:

Получите SSL сертификат (например, через Let's Encrypt):

 ```bash
 sudo apt install -y certbot
 sudo certbot certonly --standalone -d mail.your_domain.com
 ```

Обновите конфигурации Postfix и Dovecot для использования SSL:

Postfix (`/etc/postfix/main.cf`):

 ```ini
 smtpd_tls_cert_file=/etc/letsencrypt/live/mail.your_domain.com/fullchain.pem
 smtpd_tls_key_file=/etc/letsencrypt/live/mail.your_domain.com/privkey.pem
 smtpd_use_tls=yes
 ```

Dovecot (`/etc/dovecot/conf.d/10-ssl.conf`):

 ```ini
 ssl = yes
 ssl_cert = </etc/letsencrypt/live/mail.your_domain.com/fullchain.pem
 ssl_key = </etc/letsencrypt/live/mail.your_domain.com/privkey.pem
 ```

Перезапустите сервисы:

 ```bash
 sudo systemctl restart postfix
 sudo systemctl restart dovecot
 ```

### Тестирование и переключение DNS

#### Проверьте работу сайта на новом сервере:

     - Убедитесь, что сайт загружается корректно.
     - Проверьте функциональность всех сервисов (почта, очереди, задачи Cron и т.д.).

#### Обновите DNS-записи:

     - Измените A-записи вашего домена на новый IP-адрес.
     - Дождитесь обновления DNS (может занять до 24 часов).

#### Проверьте, что трафик перенаправляется на новый сервер.

### Завершение переноса

#### Обеспечьте безопасность нового сервера:

     - Настройте брандмауэр (например, UFW):
         
```bash
 sudo ufw allow OpenSSH
 sudo ufw allow 'Nginx Full'
 sudo ufw enable
 ```

     - Отключите доступ по паролю для SSH, если используете ключи.

#### Удалите или архивируйте старый сервер после успешного переноса и тестирования.

## Упрощенный вариант развертывания веб-сервера
    
1. Установите Hestia Control Panel на новом сервере:

    ```bash
    wget https://raw.githubusercontent.com/hestiacp/hestiacp/release/install/hst-install.sh
    bash hst-install.sh
    ```
   
2. Создайте новый домен и установите SSL сертификат через Hestia Control Panel.
3. Перенесите файлы и базу данных на новый сервер.
4. Настройте окружение, зависимости и оптимизацию.
5. Настройте Redis, Supervisor и Cron задачи.
6. Проверьте работу сайта и переключите DNS-записи.
7. Убедитесь, что трафик перенаправляется на новый сервер.
8. Обеспечьте безопасность нового сервера и удалите старый сервер.


Дополнительные рекомендации

- Резервное копирование: Перед началом переноса сделайте полные резервные копии файлов и базы данных.
- Документация: Ведите документацию всех выполненных шагов для будущих переносов или восстановления.
- Мониторинг: Настройте мониторинг сервера для отслеживания производительности и доступности.
- Безопасность: Регулярно обновляйте сервер и устанавливайте патчи безопасности.

Links:
- [Laravel Deploy](https://laravel.com/docs/11.x/deployment)
- [Horizon](https://laravel.com/docs/11.x/horizon#deploying-horizon)
- [Hestia Control Panel](https://hestiacp.com)
