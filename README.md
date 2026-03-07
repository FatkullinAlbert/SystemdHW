# SystemdHW
Домашнее задание на тему: "Работа с Systemd"
Задание:
Выполнить следующие задания и подготовить развёртывание результата выполнения с использованием Vagrant и Vagrant shell provisioner (или Ansible, на Ваше усмотрение):
Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
Предлагаю разделить эти задания и разбирать их по отделньости, независимо друг от друга:
Создаём файл конфигурации:
sudo cat > /etc/default/watchlog <<EOF
# Configuration for watchlog service
WORD="ALERT"
LOG=/var/log/watchlog.log
EOF
Тестовый лог файл:
sudo tee /var/log/watchlog.log <<EOF
Info: system started
ALERT: CPU temperature too high
Info: all good
EOF
Скрипт проверки:
sudo cat > /opt/watchlog.sh <<'EOF'
#!/bin/bash
WORD=$1
LOG=$2
DATE=$(date)

if grep -q "$WORD" "$LOG" 2>/dev/null; then
    logger "$DATE: I found word, Another user!"
else
    exit 0
fi
EOF
И даём права на исполнеине:
sudo chmod +x /opt/watchlog.sh
Unit‑файл сервиса:
sudo cat > /etc/systemd/system/watchlog.service <<EOF
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh \$WORD \$LOG
EOF
Unit‑файл таймера:
sudo cat > /etc/systemd/system/watchlog.timer <<EOF
[Unit]
Description=Run watchlog script every 30 seconds

[Timer]
OnUnitActiveSec=30
Unit=watchlog.service

[Install]
WantedBy=multi-user.target
EOF
Запуск и проверка:
sudo systemctl start watchlog.timer
sudo systemctl enable watchlog.timer
Установка spawn-fcgi и unit-файла из init-скрипта:
Устанавилваем необходимы пакеты:
sudo apt install -y spawn-fcgi php php-cgi php-cli
Создаём файл с переменными:
sudo cat > /etc/default/phpfastcgi <<EOF
# Parameters for spawn-fcgi php-cgi service
SERVER_IP=127.0.0.1
SERVER_PORT=9000
SERVER_USER=www-data
SERVER_GROUP=www-data
SERVER_CHILDS=5
PHP_CGI=/usr/bin/php-cgi
PIDFILE=/var/run/php-cgi.pid
EOF
Создаём Init-файл:
sudo cat > /etc/systemd/system/spawn-fcgi.service <<EOF
[Unit]
Description=PHP FastCGI Process Manager (spawn-fcgi)
After=network.target

[Service]
Type=forking
PIDFile=/var/run/php-cgi.pid
EnvironmentFile=/etc/default/phpfastcgi
ExecStart=/usr/bin/spawn-fcgi -a \${SERVER_IP} -p \${SERVER_PORT} -u \${SERVER_USER} -g \${SERVER_GROUP} -P \${PIDFILE} -C \${SERVER_CHILDS} -f \${PHP_CGI}
ExecStop=/bin/kill -QUIT \$MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
Проверяем:
sudo systemctl start spawn-fcgi
Убеждаемся, что сервис активен и процесс слушает порт 9000:
ss -tnlp | grep 9000
Теперь задание 3: Модифицировать unit в Nginx шаблон
Устанавилваем Nginx:
sudo apt install -y nginx
Создаём Init-шаблон.
sudo cat > /etc/systemd/system/nginx@.service <<'EOF'
[Unit]
Description=A high performance web server (instance %I)
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
EOF
Файлы конфигурации для 2 инстансов
1 инстанс
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx-first.conf
sudo sed -i 's/pid \/run\/nginx.pid;/pid \/run\/nginx-first.pid;/' /etc/nginx/nginx-first.conf
sudo sed -i 's/listen 80 default_server;/listen 9001;/' /etc/nginx/nginx-first.conf
2 инстанс
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx-second.conf
sudo sed -i 's/pid \/run\/nginx.pid;/pid \/run\/nginx-second.pid;/' /etc/nginx/nginx-second.conf
sudo sed -i 's/listen 80 default_server;/listen 9002;/' /etc/nginx/nginx-second.conf
Проверяем
systemctl status nginx@first
systemctl status nginx@second
Если они работают, то всё отлично. Если до этого работали с Nginx, то можно закоментировть include sites-enabled/* (иначе вполне вероятно не заработает).
