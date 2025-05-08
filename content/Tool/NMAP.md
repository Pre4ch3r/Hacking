---
title: NMAP
created: 2025-05-08 10:17
updated: quinta-feira 8º maio 2025 10:17:14
tags: [nmap, network, tool]
draft: false
---

# NMAP

O **Nmap (Network Mapper)** é uma das ferramentas mais populares e poderosas de **varredura e auditoria de redes**. Ele é usado tanto por administradores de sistemas quanto por analistas de segurança para mapear redes, identificar dispositivos, serviços e sistemas operacionais, além de detectar vulnerabilidades.

---

## 🔍 O que é o Nmap?

- **Nmap** é uma ferramenta de linha de comando (com interface gráfica opcional: **Zenmap**) usada para **varredura de portas**, **detecção de hosts ativos**, **identificação de sistemas operacionais**, e **auditorias de segurança**.
    
- Desenvolvido por **Gordon Lyon (Fyodor)**, o Nmap é open-source e amplamente utilizado em testes de penetração.
    

---

## 🎯 Para que serve o Nmap?

Ele pode ser usado para:

- Descobrir **hosts ativos** em uma rede.
    
- Verificar quais **portas estão abertas** e quais **serviços** estão sendo executados.
    
- Tentar identificar o **sistema operacional** e o **uptime** do host.
    
- Realizar **detecção de firewalls**, IDS (Intrusion Detection Systems) e IPS (Intrusion Prevention Systems).
    
- Auxiliar na **evasão de mecanismos de segurança** para testes controlados.
    

---

## 🧪 Exemplos práticos de varredura com o Nmap

### 1. **Varredura rápida de portas TCP padrão**

```bash
nmap 192.168.0.1
```

> Faz uma varredura nas 1.000 portas TCP mais comuns.

---

### 2. **Varredura completa de todas as 65.535 portas TCP**

```bash
nmap -p- 192.168.0.1
```

---

### 3. **Detecção de sistema operacional e serviços**

```bash
nmap -A 192.168.0.1
```

> Ativa detecção de OS, versão de serviços, script scanning e traceroute.

---

### 4. **Varredura silenciosa para evitar alertas (evasão de IDS/IPS)**

#### a) **Alterar tempo entre pacotes (slow scan)**

```bash
nmap -T2 192.168.0.1
```

> Diminui a velocidade da varredura, dificultando a detecção.

#### b) **Fragmentação de pacotes (IDS evasão clássica)**

```bash
nmap -f 192.168.0.1
```

> Fragmenta os pacotes em partes menores, confundindo IDS mais simples.

#### c) **Spoofando o endereço MAC**

```bash
nmap --spoof-mac 00:11:22:33:44:55 192.168.0.1
```

> Finge ser outro dispositivo de rede.

#### d) **Scan com source port comum (como 53 - DNS)**

```bash
nmap --source-port 53 192.168.0.1
```

> Alguns firewalls permitem portas "confiáveis" como 53, 20, 443 etc.

#### e) **Scan em modo decoy (usando IPs falsos)**

```bash
nmap -D 192.168.0.100,192.168.0.200,ME 192.168.0.1
```

> Confunde o IDS usando IPs falsos junto com o seu.

#### f) **Scan Idle (com host zumbi)**

```bash
nmap -sI 192.168.0.99 192.168.0.1
```

> Usando um host "zumbi" para evitar exposição do seu próprio IP.

---

## ⚠️ Aviso de ética e legalidade

Ferramentas como o Nmap são extremamente úteis, mas também podem ser perigosas se usadas sem autorização. Usar o Nmap para escanear redes ou sistemas sem permissão **é ilegal** e pode resultar em **sérias consequências legais**.

---

## 🧠 O que é `--script`?

A opção `--script` do **Nmap** ativa os **Nmap Scripting Engine (NSE) scripts**, que são scripts poderosos escritos em **Lua** para realizar varreduras mais avançadas — desde detecção de vulnerabilidades até autenticação em serviços e enumeração de usuários.

- O NSE permite executar **scripts customizados** em fases específicas do scan.
    
- Os scripts estão organizados em **categorias**, como:
    
    - `auth`: scripts de autenticação.
        
    - `vuln`: scripts de detecção de vulnerabilidades.
        
    - `default`: scripts executados com a opção `-sC`.
        
    - `safe`: considerados seguros para uso regular.
        
    - `intrusive`: scripts que podem causar impacto.
        

---

## 🔧 Sintaxe básica

```bash
nmap --script=<nome_do_script> <alvo>
```

Você também pode usar categorias ou expressões:

```bash
nmap --script=vuln 192.168.0.1
```

---

## 🧪 Exemplos práticos com `--script`

### 1. **Scan com scripts padrão**

```bash
nmap -sC 192.168.0.1
```

> Equivalente a `--script=default`.

---

### 2. **Enumeração de usuários em SMB**

```bash
nmap -p 445 --script=smb-enum-users 192.168.0.1
```

---

### 3. **Detecção de vulnerabilidades comuns**

```bash
nmap --script=vuln 192.168.0.1
```

> Roda todos os scripts da categoria `vuln`, como `smb-vuln-ms17-010`.

---

### 4. **Verificar se o alvo é vulnerável à Shellshock**

```bash
nmap --script=http-shellshock --script-args uri=/cgi-bin/test.cgi 192.168.0.1
```

---

### 5. **Detectar diretórios ocultos em um servidor web**

```bash
nmap -p 80 --script=http-enum 192.168.0.1
```

---

### 6. **Explorar backdoor no MySQL**

```bash
nmap -p 3306 --script=mysql-empty-password 192.168.0.1
```

---

### 7. **Verificar serviços com SSL fraco**

```bash
nmap --script=ssl-enum-ciphers -p 443 192.168.0.1
```

---

### 8. **Buscar informações DNS de um alvo**

```bash
nmap --script=dns-brute 192.168.0.1
```

---

## 🔍 Como listar todos os scripts disponíveis?

```bash
ls /usr/share/nmap/scripts/
```

Ou use:

```bash
nmap --script-help=default
```

Ou para ver detalhes de um script específico:

```bash
nmap --script-help=http-enum
```

---

### 💡 Dica avançada: Rodar múltiplos scripts com filtros

```bash
nmap --script="(default or intrusive) and not dos" 192.168.0.1
```

> Executa scripts padrão ou intrusivos, mas **exclui** os que causam negação de serviço (DoS).

---


