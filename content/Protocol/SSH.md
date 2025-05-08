---
title: SSH
created: 2025-05-08 10:32
updated: quinta-feira 8º maio 2025 10:32:48
tags: [ssh, protocol, network, tool]
draft: true
---

# SSH

O **SSH (Secure Shell)** é um protocolo de rede utilizado para acessar e gerenciar dispositivos remotamente de forma **segura**. Ele substitui protocolos mais antigos e inseguros como o Telnet, oferecendo **criptografia**, **autenticação** e **integridade dos dados** durante a comunicação.

---

## 🔐 O que é SSH?

- **SSH** significa **Secure Shell**.
    
- Ele permite **acesso remoto** a sistemas e servidores usando uma **interface de terminal**.
    
- Fornece um **canal seguro** sobre uma rede insegura (como a internet).
    
- Opera normalmente na porta **22/tcp**.
    

---

## 🎯 Para que serve o SSH?

- Acessar servidores remotos via linha de comando (como servidores Linux).
    
- Copiar arquivos de forma segura com **SCP** ou **SFTP**.
    
- Executar comandos remotos.
    
- Encaminhar conexões com **túneis SSH** (port forwarding).
    
- Criar **túneis VPN-like** para criptografar tráfego.
    

---

## 🧱 Como o SSH funciona?

1. O cliente (por exemplo, `ssh user@192.168.0.1`) tenta se conectar ao servidor.
    
2. O servidor envia sua **chave pública**.
    
3. O cliente verifica a chave e começa uma negociação criptografada.
    
4. O usuário faz login com **senha** ou **chave SSH privada**.
    
5. Após autenticação, a sessão é iniciada com criptografia fim a fim.
    

---

## 🔐 Autenticação no SSH

Existem duas formas principais:

### 1. **Senha**

- O usuário digita sua senha para se autenticar.
    

### 2. **Chave pública/privada**

- O usuário gera um par de chaves (`ssh-keygen`) e envia a **chave pública** ao servidor (`~/.ssh/authorized_keys`).
    
- A autenticação é feita com a **chave privada** local.
    

---

## 📦 Exemplos de uso do SSH

### 1. **Conectar-se a um servidor**

```bash
ssh usuario@192.168.0.1
```

### 2. **Usar uma porta diferente**

```bash
ssh -p 2222 usuario@192.168.0.1
```

### 3. **Copiar arquivos com SCP**

```bash
scp arquivo.txt usuario@192.168.0.1:/home/usuario/
```

### 4. **Redirecionamento de porta (port forwarding)**

```bash
ssh -L 8080:localhost:80 usuario@192.168.0.1
```

> Encaminha sua porta 8080 local para a 80 do servidor via túnel SSH.

### 5. **Login com chave**

```bash
ssh -i ~/.ssh/minha-chave usuario@192.168.0.1
```

---

## 🛡️ Segurança no SSH

- Utilize **autenticação por chave pública**, desativando senhas (`PasswordAuthentication no`).
    
- Use **fail2ban** ou outras ferramentas para evitar brute-force.
    
- Altere a porta padrão (22) para uma porta alta.
    
- Desative o login de root direto (`PermitRootLogin no`).
    

---


## ✅ Parte 1: Gerar e usar chave SSH (lado do cliente)

### 🔧 Passo 1: Gerar par de chaves (cliente)

No seu terminal (Linux/macOS ou Git Bash no Windows), execute:

```bash
ssh-keygen -t rsa -b 4096 -C "seu@email.com"
```

- Pressione Enter para aceitar o local padrão (`~/.ssh/id_rsa`).
    
- Você pode definir uma senha para proteger a chave privada (opcional, mas recomendado).
    

### 🗂 Resultado:

- **~/.ssh/id_rsa** → chave privada (guarde com segurança).
    
- **~/.ssh/id_rsa.pub** → chave pública (para copiar no servidor).
    

---

## ✅ Parte 2: Enviar chave pública para o servidor

Se você já tem acesso SSH com senha, execute:

```bash
ssh-copy-id usuario@ip_do_servidor
```

Ou manualmente:

```bash
scp ~/.ssh/id_rsa.pub usuario@ip_do_servidor:/tmp/
ssh usuario@ip_do_servidor
mkdir -p ~/.ssh
cat /tmp/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
rm /tmp/id_rsa.pub
```

---

## ✅ Parte 3: Testar acesso com chave

Agora, tente se conectar sem usar senha:

```bash
ssh usuario@ip_do_servidor
```

Se você configurou corretamente, ele usará sua chave privada.

---

## 🛠️ Parte 4: Configurar o servidor SSH (Linux)

### Editar o arquivo de configuração:

```bash
sudo nano /etc/ssh/sshd_config
```

Ajuste ou verifique os seguintes parâmetros:

```bash
Port 22                    # ou altere para outra porta
PermitRootLogin no         # evita login direto como root
PasswordAuthentication no  # desativa login por senha
PubkeyAuthentication yes   # garante que login por chave está ativo
```

### Reinicie o serviço SSH:

```bash
sudo systemctl restart ssh
```

### (Opcional) Alterar porta para reduzir tentativas de ataque:

```bash
Port 2222
```

Lembre-se de ajustar seu firewall (ex: `ufw allow 2222`).

---

## 🧪 Testar com a nova porta:

```bash
ssh -p 2222 usuario@ip_do_servidor
```

---

## Autenticação via certificados

A **autenticação via certificados** no SSH é diferente da autenticação com **chaves públicas tradicionais** (`id_rsa` ou `id_ed25519`). Com certificados, uma autoridade (normalmente você mesmo ou uma CA interna) **assina uma chave pública**, e o servidor SSH só confia em chaves assinadas por essa autoridade.

Isso permite uma **gestão centralizada de acesso SSH** — muito útil para ambientes corporativos com muitos usuários e servidores.

---

## 🧾 Conceitos-chave

- 🔑 **Chave pública do usuário** → como no SSH tradicional.
    
- 📄 **Certificado SSH** → a chave pública assinada por uma **CA SSH**.
    
- 🛡️ **CA (Certificate Authority)** → entidade que assina as chaves.
    
- 🔐 O servidor SSH aceita apenas conexões cujas **chaves foram assinadas** por uma CA confiável.
    

---

## 🛠️ Como funciona na prática

### ✅ 1. Criar uma CA (autoridade certificadora)

No computador da CA (pode ser o seu):

```bash
ssh-keygen -f ~/.ssh/ssh_ca -C "Minha CA SSH"
```

Isso gera:

- `ssh_ca` (chave privada da CA) — **NUNCA compartilhar**
    
- `ssh_ca.pub` (chave pública da CA) — usada nos servidores
    

---

### ✅ 2. Criar uma chave pública do usuário

No computador do usuário (cliente):

```bash
ssh-keygen -f ~/.ssh/id_ed25519 -C "usuario@empresa"
```

---

### ✅ 3. Assinar a chave pública com a CA

No computador da CA:

```bash
ssh-keygen -s ~/.ssh/ssh_ca -I usuario-cert -n usuario -V +52w ~/.ssh/id_ed25519.pub
```

**Explicação:**

- `-s`: chave privada da CA
    
- `-I`: identificador do certificado
    
- `-n`: nome do usuário válido para essa chave
    
- `-V`: validade (por exemplo, `+52w` = 1 ano)
    

Resultado: um certificado SSH chamado `id_ed25519-cert.pub`.

---

### ✅ 4. Configurar o servidor SSH para confiar na CA

No **servidor SSH**:

1. Copie a **chave pública da CA** para o servidor:
    

```bash
scp ~/.ssh/ssh_ca.pub usuario@servidor:/etc/ssh/
```

2. Edite o `/etc/ssh/sshd_config` e adicione:
    

```bash
TrustedUserCAKeys /etc/ssh/ssh_ca.pub
```

3. Reinicie o serviço SSH:
    

```bash
sudo systemctl restart ssh
```

---

### ✅ 5. Conectar usando o certificado

No cliente, use:

```bash
ssh -i ~/.ssh/id_ed25519 usuario@servidor
```

O SSH usará automaticamente `id_ed25519-cert.pub` se ele estiver presente ao lado da chave privada.

---

## 📌 Resumo visual do fluxo

```plaintext
[Usuário]
   |
   | cria chave → id_ed25519.pub
   |
[CA]
   |
   | assina chave → id_ed25519-cert.pub
   |
[Servidor]
   |
   | confia na CA → ssh_ca.pub
   |
[Usuário] → conecta com certificado → acesso autorizado
```

---


