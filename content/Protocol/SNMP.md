---
title: SNMP
created: 2025-05-08 09:59
tags: [snmp, protocol, network]
draft: false
---

# SNMP

O **SNMP (Simple Network Management Protocol)** é um protocolo da camada de aplicação usado para gerenciar e monitorar dispositivos em redes IP, como roteadores, switches, servidores, impressoras e outros dispositivos habilitados para rede.

### 🧩 Como o SNMP funciona?

O SNMP opera com uma arquitetura baseada em três componentes principais:

1. **Gerente (Manager)**:
    
    - Normalmente um software (como um NMS – Network Management System) que roda em um servidor.
        
    - Ele envia solicitações e recebe dados dos agentes.
        
    - Exemplo: Pede ao roteador qual é a taxa de utilização da CPU.
        
2. **Agente (Agent)**:
    
    - Software que roda no dispositivo gerenciado (por exemplo, um switch ou servidor).
        
    - Ele coleta e armazena informações sobre o dispositivo e responde às solicitações do gerente.
        
    - Pode enviar alertas (traps) para o gerente quando algo importante acontece (por exemplo, uma falha de interface).
        
3. **MIB (Management Information Base)**:
    
    - Um banco de dados estruturado que define as informações que podem ser coletadas via SNMP.
        
    - Cada dado é acessado por um identificador único chamado **OID (Object Identifier)**.
        

---

### 🔄 Operações básicas do SNMP:

- **GET**: O gerente pede um valor ao agente.
    
- **SET**: O gerente altera um valor no agente.
    
- **GETNEXT**: O gerente pede o próximo valor na MIB (usado para percorrer listas).
    
- **TRAP/INFORM**: O agente envia uma notificação ao gerente de forma não solicitada (ex: erro de hardware).
    

---

### 🔒 Versões do SNMP:

1. **SNMPv1**: Primeira versão, simples, mas com segurança fraca (envio em texto claro).
    
2. **SNMPv2c**: Versão melhorada, mas ainda com autenticação baseada em "community string".
    
3. **SNMPv3**: Versão mais segura, com suporte a autenticação e criptografia.
    

---

### 📦 Exemplo prático:

Um gerente SNMP pode perguntar a um switch:

> "Qual é o tráfego na interface eth0?"

E o agente responderá:

> "O tráfego atual é 123456 bytes."

---

Vamos aprofundar com um exemplo prático de como usar o SNMP para coletar informações de um dispositivo de rede, utilizando o comando `snmpgetnext` da suíte Net-SNMP.

---

### 🔍 Exemplo prático com o comando `snmpgetnext`

O comando `snmpgetnext` é utilizado para obter o próximo valor na árvore MIB (Management Information Base) de um dispositivo. Ele é útil para percorrer sequencialmente os objetos gerenciados.

#### Exemplo de uso:

```bash
snmpgetnext -v 2c -c public demo.net-snmp.org system.sysUpTime.0
```

**Explicação dos parâmetros:**

- `-v 2c`: Especifica a versão do SNMP (neste caso, SNMPv2c).
    
- `-c public`: Define a "community string" (senha) para autenticação.
    
- `demo.net-snmp.org`: Endereço do agente SNMP.
    
- `system.sysUpTime.0`: O OID (Object Identifier) inicial para a consulta.
    

**Saída esperada:**

```bash
system.sysContact.0 = STRING: Net-SNMP Coders
```

Neste exemplo, o comando retorna o próximo objeto após `system.sysUpTime.0`, que é `system.sysContact.0`.

#### Percorrendo a árvore MIB:

Você pode continuar utilizando o `snmpgetnext` para percorrer a árvore MIB:

```bash
snmpgetnext -v 2c -c public demo.net-snmp.org system.sysContact.0
```

E assim por diante, até atingir o final da árvore MIB.

---

### 🔐 Exemplo com SNMPv3 (mais seguro)

O SNMPv3 oferece recursos aprimorados de segurança, como autenticação e criptografia. Aqui está um exemplo de como utilizá-lo:

```bash
snmpgetnext -v 3 -l authPriv -u MD5User -a MD5 -A "senhaAutenticacao" -x DES -X "senhaPrivacidade" demo.net-snmp.org system.sysUpTime.0
```

**Explicação dos parâmetros:**

- `-v 3`: Especifica a versão do SNMP (SNMPv3).
    
- `-l authPriv`: Define o nível de segurança (autenticação e privacidade).
    
- `-u MD5User`: Nome de usuário para autenticação.
    
- `-a MD5`: Algoritmo de autenticação (MD5).
    
- `-A "senhaAutenticacao"`: Senha de autenticação.
    
- `-x DES`: Algoritmo de criptografia (DES).
    
- `-X "senhaPrivacidade"`: Senha de criptografia.
    
- `demo.net-snmp.org`: Endereço do agente SNMP.
    
- `system.sysUpTime.0`: OID inicial para a consulta.
    

**Saída esperada:**

```bash
system.sysUpTime.0 = Timeticks: (83467131) 9 days, 15:51:11.31
```

Este comando retorna o tempo de atividade do sistema com segurança aprimorada.

---

### 🛠️ Ferramenta complementar: `snmpwalk`

Para facilitar a exploração da árvore MIB, você pode utilizar o comando `snmpwalk`, que executa uma série de `snmpgetnext` automaticamente:

```bash
snmpwalk -v 2c -c public demo.net-snmp.org system
```

Este comando retornará todos os objetos sob o OID `system`, permitindo uma visão abrangente das informações disponíveis.

---

Vamos explorar como utilizar a ferramenta **snmp-check** para realizar uma enumeração SNMP em dispositivos de rede. Essa ferramenta é útil para coletar informações detalhadas de dispositivos como roteadores, switches, servidores e impressoras que possuem o SNMP habilitado.

---

### 🔧 Instalação do snmp-check

O **snmp-check** está disponível por padrão em distribuições como o Kali Linux. Caso não esteja instalado, você pode instalá-lo com o seguinte comando:([Knowledge Base (KB)](https://kb.offsec.nl/tools/other/snmp-check/))

```bash
sudo apt install snmp-check
```

---

### ⚙️ Uso básico do snmp-check

A sintaxe básica do comando é:

```bash
snmp-check [OPÇÕES] <endereço IP do alvo>
```

Algumas opções úteis incluem:

- `-c <comunidade>`: Define a string de comunidade SNMP (padrão: `public`).
    
- `-v <versão>`: Especifica a versão do SNMP (1 ou 2c; padrão: 1).
    
- `-p <porta>`: Define a porta SNMP (padrão: 161).
    
- `-t <tempo>`: Define o tempo limite em segundos (padrão: 5).
    
- `-r <tentativas>`: Define o número de tentativas de solicitação (padrão: 1).
    
- `-w`: Detecta acesso de gravação (separado por enumeração).
    
- `-d`: Desativa a enumeração de conexões TCP.
    
- `-i`: Exibe a versão do script.
    
- `-h`: Exibe o menu de ajuda.([Hacking Reviews](https://www.hacking.reviews/2017/11/snmpcheck-tool-to-enumerate-information.html), [nodeping.com](https://nodeping.com/snmp_check.html))
    

---

### 🧪 Exemplo prático

Para realizar uma enumeração básica em um dispositivo com o IP `192.168.1.100`, utilizando a comunidade `public` e a versão SNMPv2c, execute:

```bash
snmp-check -c public -v 2c 192.168.1.100
```

Isso retornará informações como:([IEMLabs](https://iemlabs.com/blogs/snmp-check/))

- Informações do sistema (hostname, descrição, contato, localização).
    
- Tempo de atividade do sistema.
    
- Informações de rede (interfaces de rede, IPs, rotas).
    
- Conexões TCP e portas UDP em escuta.
    
- Contas de usuário.
    
- Informações de hardware e armazenamento.
    
- Serviços de rede em execução.([Knowledge Base (KB)](https://kb.offsec.nl/tools/other/snmp-check), [IEMLabs](https://iemlabs.com/blogs/snmp-check/))
    

Exemplo de saída:

```
[*] Informações do Sistema:
  Endereço IP do Host: 192.168.1.100
  Nome do Host: Router01
  Descrição: Roteador Cisco 2900
  Contato: admin@empresa.com
  Localização: Sala de servidores
  Tempo de atividade do sistema: 3 dias, 12 horas, 45 minutos
  Data do sistema: 2025-05-08 09:30:00
```

Para visualizar informações específicas, como interfaces de rede, utilize:

```bash
snmp-check -c public -v 2c -i 192.168.1.100
```

Isso exibirá detalhes sobre as interfaces de rede do dispositivo.

---

### 🛡️ Considerações de segurança

O uso do **snmp-check** deve ser realizado com responsabilidade e autorização adequada, especialmente em ambientes de produção. A enumeração SNMP pode gerar alertas em sistemas de detecção de intrusões (IDS) e afetar o desempenho da rede. Além disso, é importante garantir que o SNMP esteja configurado de forma segura, utilizando versões mais recentes (como SNMPv3) e restringindo o acesso à comunidade SNMP.

---


