---
title: Feroxbuster
created: 2025-11-03 15:01
tags:
draft: false
---

# Feroxbuster

Ferox é a abreviação de _Ferric Oxide_. Ferric Oxide, de forma simples, é ferrugem em inglês. O nome _rustbuster_ já estava sendo usado, então o autor decidiu por uma variação. 

E o que ele faz? feroxbuster é uma ferramenta projetada para realizar _Forced Browsing_.

O _Forced Browsing_ é um ataque cujo objetivo é enumerar e acessar recursos que não são referenciados pela aplicação web, mas que ainda são acessíveis por um atacante.

feroxbuster usa força bruta combinada com uma lista de palavras para buscar conteúdo não vinculado em diretórios de destino. Esses recursos podem armazenar informações sensíveis sobre aplicações web e sistemas operacionais, como código-fonte, credenciais, endereçamento de rede interna, etc...

Este ataque também é conhecido como _Predictable Resource Location_, _File Enumeration_, _Directory Enumeration_ e _Resource Enumeration_.

Baixe o `feroxbuster_amd64.deb` da seção **Releases**. Após isso, use seu gerenciador de pacotes favorito para instalar o `.deb`.

```bash
curl -sLO https://github.com/epi052/feroxbuster/releases/latest/download/feroxbuster_amd64.deb.zip
unzip feroxbuster_amd64.deb.zip
sudo apt install ./feroxbuster_*_amd64.deb
```

**Instalação no Kali**  

O `feroxbuster` foi recentemente adicionado aos repositórios oficiais do Kali Linux.

Se você está usando o Kali, esse é o método de instalação recomendado. Instalar a partir dos repositórios adiciona um arquivo `ferox-config.toml` em `/etc/feroxbuster/`, adiciona completions de comandos para `bash`, `fish` e `zsh`, inclui uma entrada de página `man` e instala o próprio `feroxbuster`.

```bash
sudo apt update && sudo apt install -y feroxbuster
```

## Recursos Principais

Presentes desde o primeiro dia

### Múltiplos Valores

Opções que aceitam múltiplos valores são muito flexíveis. Considere as seguintes formas de especificar extensões:

```bash
./feroxbuster -u http://127.1 -x pdf -x js,html -x php txt json,docx
```

O comando acima adiciona `.pdf`, `.js`, `.html`, `.php`, `.txt`, `.json` e `.docx` a cada URL.

Todos os métodos acima (múltiplas flags, separadas por espaço, separadas por vírgula, etc...) são válidos e intercambiáveis. O mesmo se aplica a URLs, headers, códigos de status, queries e filtros de tamanho.

### Incluir Headers

```bash
./feroxbuster -u http://127.1 -H Accept:application/json "Authorization: Bearer {token}"
```

**Nota:** para incluir um header contendo uma vírgula, use `ferox-config.toml` (discussão relacionada)

### IPv6, varredura não-recursiva com logging em nível INFO habilitado

```bash
./feroxbuster -u http://[::1] --no-recursion -vv
```

### Ler URLs de STDIN; canalizar apenas URLs resultantes para outra ferramenta

```bash
cat targets | ./feroxbuster --stdin --silent -s 200 301 302 --redirects -x js | fff -s 200 -o js-files
```

### Rotear tráfego através do Burp

```bash
./feroxbuster -u http://127.1 --insecure --proxy http://127.0.0.1:8080
```

### Rotear tráfego através de um proxy SOCKS (incluindo buscas DNS)

```bash
./feroxbuster -u http://127.1 --proxy socks5h://127.0.0.1:9050
```

### Passar token de autenticação via parâmetro de query

```bash
./feroxbuster -u http://127.1 --query token=0123456789ABCDEF
```