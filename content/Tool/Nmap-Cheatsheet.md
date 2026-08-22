---
title: Nmap-Cheatsheet
created: 2026-08-21 19:06
tags:
  - nmap
  - network
  - tool
  - cheatsheet
draft: false
---


# Nmap Cheatsheet

Lista de exemplos práticos de comandos para varredura, evasão e uso do NSE.

## Varredura de Portas e Hosts
- **Varredura rápida de portas TCP padrão**

```bash
  nmap -T4 --top-ports 1000 <alvo>
```

_(Faz uma varredura nas 1.000 portas TCP mais comuns)_

- **Varredura completa (todas as portas)**


```bash
nmap -p- <alvo>
```

_(Varre todas as 65.535 portas TCP)_

- **Detecção completa (OS, Versão, Scripts, Traceroute)**

```
nmap -A <alvo>
```

_(Ativa detecção de OS, versão de serviços, script scanning e traceroute)_


## Evasão de Segurança (Stealth & IDS/IPS)

Técnicas para evitar detecção durante testes autorizados:

- **Scan lento (Slow scan)**: `nmap --scan-delay <tempo> <alvo>` (Diminui a velocidade para dificultar a detecção).
- **Fragmentação de pacotes**: `nmap -f <alvo>` (Confunde IDS simples).
- **Spoofing de MAC**: `nmap --spoof-mac <MAC> <alvo>` (Finge ser outro dispositivo).
- **Source Port comum**: `nmap --source-port 53 <alvo>` (Usa portas "confiáveis" como 53, 20, 443).
- **Modo Decoy**: `nmap -D RND:10 <alvo>` (Usa IPs falsos para confundir o IDS).
- **Scan Idle (Zumbi)**: `nmap -sI <zumbi> <alvo>` (Usa um host "zumbi" para não expor seu IP).

## Uso do NSE (`--script`)

Comandos específicos para scripts:

- **Scan com scripts padrão**

```
 nmap --script=default <alvo>
```
 
_(Equivalente a `-sC`)_

- **Enumeração de usuários em SMB**

```
nmap --script=smb-enum-users <alvo>
```

- **Detecção de vulnerabilidades comuns**

```
 nmap --script=vuln <alvo>
```

_(Roda scripts como `smb-vuln-ms17-010`)_

- **Verificar vulnerabilidade Shellshock**

```
nmap --script=http-shellshock <alvo>
```

- **Detectar diretórios ocultos (Web)**

```
nmap --script=http-enum <alvo>
```

- **Explorar backdoor no MySQL**

```
nmap --script=mysql-vuln-cve2012-2122 <alvo>
```

- **Verificar serviços com SSL fraco**

```
nmap --script=ssl-enum-ciphers <alvo>
```

- **Buscar informações DNS**

```
nmap --script=dns-brute <alvo>
```

## Gerenciamento de Scripts

- **Listar todos os scripts disponíveis**:

```
 ls /usr/share/nmap/scripts/
``` 

- **Ver detalhes de um script específico**:

```
nmap --script-help <nome-do-script>
```

## Filtros Avançados

Executar scripts padrão ou intrusivos, mas **excluir** os que causam negação de serviço (DoS):

```
nmap --script "default or intrusive" --script-args unsafe=0 <alvo>
```

## Ver também

- Conceitos gerais e NSE: [[NMAP]]
