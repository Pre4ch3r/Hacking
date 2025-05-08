---
title: Daloradius
created: 2025-05-08 11:01
updated: quinta-feira 8º maio 2025 11:01:05
tags: [daloradius, tool, server]
draft: true
---

# Daloradius

---

## 🔎 O que é o daloRADIUS?

O **daloRADIUS** é uma **interface web** baseada em PHP para gerenciamento de servidores **FreeRADIUS**. Ele oferece uma maneira gráfica e mais acessível de gerenciar autenticações de rede, contas de usuários, perfis, grupos e estatísticas.

- Ele funciona como **front-end** do **FreeRADIUS**, que é um servidor de autenticação, autorização e contabilização (AAA) para redes.
    
- Normalmente é usado em conjunto com **bancos de dados MySQL/MariaDB**.
    

---

## 🎯 Para que serve o daloRADIUS?

O daloRADIUS serve principalmente para:

- Criar, editar e excluir **usuários e grupos**.
    
- Definir **limites de tempo**, **velocidade de banda**, **número de conexões simultâneas**.
    
- Integrar com **captive portals** (ex: para Wi-Fi de hotéis ou cafés).
    
- Gerar **relatórios de uso**, sessões, logs de autenticação e erros.
    
- Monitorar **contabilidade (accounting)** de uso da rede.
    
- Oferecer **administração multiusuário** com diferentes permissões.
    

---

## ✅ Vantagens do uso do daloRADIUS

|Vantagem|Descrição|
|---|---|
|🖥️ Interface Web|Facilita o uso por administradores sem experiência em linha de comando.|
|📊 Estatísticas e gráficos|Oferece relatórios visuais de conexões e uso de rede.|
|👥 Gerenciamento de usuários|Criação de contas, limites de acesso e perfis personalizados.|
|🧩 Integração|Funciona bem com captive portals (ex: CoovaChilli, pfSense, MikroTik).|
|🔐 Suporte a métodos de autenticação|Inclui PAP, CHAP, MSCHAPv2, EAP, entre outros.|
|🗂️ Banco de dados|Permite backend MySQL para fácil consulta e integração.|

---

## ⚠️ Riscos de segurança comuns no daloRADIUS

Como qualquer aplicação web, o daloRADIUS **precisa ser corretamente configurado** para evitar riscos. Alguns dos problemas mais comuns incluem:

### 1. **Credenciais padrão ou fracas**

- O painel web pode ser comprometido se o admin esquecer de alterar `administrator:radius` ou usar senhas fracas.
    

### 2. **Conexões não criptografadas (HTTP ao invés de HTTPS)**

- Se acessado via HTTP, senhas e dados de usuários podem ser interceptados (MITM).
    

### 3. **Exposição à internet**

- Deixar o daloRADIUS acessível publicamente sem firewall/VPN aumenta risco de ataques.
    

### 4. **SQL Injection**

- Se não atualizado, versões antigas podem ser vulneráveis a injeções SQL.
    

### 5. **Falta de isolamento do banco de dados**

- Banco de dados usado pelo daloRADIUS pode ser acessível por usuários não autorizados.
    

### 6. **Permissões excessivas no MySQL**

- O usuário do banco usado pelo daloRADIUS pode ter privilégios desnecessários (como DROP DATABASE).
    

### 7. **Logs sensíveis**

- Informações de autenticação podem ser armazenadas em logs sem proteção adequada.
    

---

## 🔐 Boas práticas de segurança para o daloRADIUS

- **Use HTTPS** (SSL/TLS) no servidor web.
    
- **Altere as credenciais padrão** imediatamente.
    
- **Restrinja o acesso** ao painel web por IP ou VPN.
    
- **Atualize o daloRADIUS e o FreeRADIUS** regularmente.
    
- **Use firewall** para limitar conexões externas à porta do daloRADIUS (geralmente 80/443).
    
- **Faça backups regulares** do banco de dados e configurações.
    
- **Revise permissões** do banco de dados e do sistema de arquivos.
    

---

Vamos montar um **ambiente com FreeRADIUS + daloRADIUS** em um servidor Linux. Abaixo está um **guia passo a passo** para instalar e configurar tudo em uma máquina com **Ubuntu Server 20.04+** (funciona em outras distros com ajustes).

---

## 🏗️ Objetivo:

- Instalar o **FreeRADIUS**
    
- Configurar o **MySQL/MariaDB** como backend
    
- Instalar e configurar o **daloRADIUS**
    
- Ter uma **interface web para gerenciar usuários e autenticações**
    

---

## ✅ Pré-requisitos

Você precisa de:

- Uma máquina com **Ubuntu Server**
    
- Acesso com **sudo/root**
    
- Servidor web (Apache) e PHP
    
- Conexão com a internet
    

---

## 🚀 Passo a Passo

### 🔹 1. Atualize o sistema

```bash
sudo apt update && sudo apt upgrade -y
```

---

### 🔹 2. Instale FreeRADIUS + MariaDB + Apache + PHP

```bash
sudo apt install freeradius freeradius-mysql mariadb-server apache2 php php-mysql php-cli php-gd php-curl php-pear php-db php-mbstring unzip git -y
```

---

### 🔹 3. Configure o banco de dados

#### Acesse o MySQL:

```bash
sudo mysql -u root
```

#### Crie banco e usuário:

```sql
CREATE DATABASE radius;
GRANT ALL PRIVILEGES ON radius.* TO 'radius'@'localhost' IDENTIFIED BY 'senhaforte';
FLUSH PRIVILEGES;
EXIT;
```

---

### 🔹 4. Popule o banco com o schema do FreeRADIUS

```bash
sudo mysql -u root radius < /etc/freeradius/3.0/mods-config/sql/main/mysql/schema.sql
```

---

### 🔹 5. Configure o FreeRADIUS para usar SQL

Edite o arquivo:

```bash
sudo nano /etc/freeradius/3.0/mods-available/sql
```

Altere:

```ini
driver = "rlm_sql_mysql"
dialect = "mysql"
server = "localhost"
port = 3306
login = "radius"
password = "senhaforte"
radius_db = "radius"
```

Ative o módulo:

```bash
sudo ln -s /etc/freeradius/3.0/mods-available/sql /etc/freeradius/3.0/mods-enabled/
```

Reinicie o serviço:

```bash
sudo systemctl restart freeradius
```

---

### 🔹 6. Baixe e instale o daloRADIUS

```bash
cd /var/www/html
sudo git clone https://github.com/lirantal/daloradius.git
sudo mv daloradius /var/www/html/daloradius
cd /var/www/html/daloradius
sudo mysql -u root radius < contrib/db/fr2-mysql-daloradius-and-freeradius.sql
sudo mysql -u root radius < contrib/db/mysql-daloradius.sql
```

---

### 🔹 7. Configurar o Apache

Dê permissões:

```bash
sudo chown -R www-data:www-data /var/www/html/daloradius
```

Crie um virtualhost se quiser, ou acesse via `http://ip_do_servidor/daloradius`.

---

### 🔹 8. Configure o arquivo de conexão ao banco

Edite:

```bash
sudo nano /var/www/html/daloradius/library/daloradius.conf.php
```

Altere:

```php
$configValues['CONFIG_DB_USER'] = 'radius';
$configValues['CONFIG_DB_PASS'] = 'senhaforte';
$configValues['CONFIG_DB_NAME'] = 'radius';
```

---

### 🔹 9. Acesse o daloRADIUS

Navegue até:  
🔗 **http://IP_DO_SERVIDOR/daloradius**

Login padrão:

- **Usuário**: `administrator`
    
- **Senha**: `radius`
    

---

## 🛡️ Dicas de segurança

- Altere a senha do admin no primeiro login.
    
- Configure HTTPS com Let’s Encrypt ou um certificado próprio.
    
- Restrinja o acesso ao painel por IP ou com autenticação básica do Apache.
    
- Faça backups do banco de dados regularmente.
    

---


