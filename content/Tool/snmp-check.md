---
title: snmp-check
created: 2025-05-08 11:13
updated: quinta-feira 8º maio 2025 11:13:41
tags: [snmp-check, network, udp, tool, concept]
draft: false
---

# snmp-check

Vamos explorar a ferramenta **`snmp-check`**, muito usada em **auditorias de segurança** para **coletar informações via SNMP** (Simple Network Management Protocol), que é amplamente utilizado em redes para monitoramento de dispositivos como switches, roteadores, impressoras, servidores etc.

---

## 🧠 O que é o `snmp-check`?

- O `snmp-check` é uma ferramenta de **enumeração SNMP em modo leitura**.
    
- Ele interage com o protocolo **SNMPv1** e **SNMPv2c** para coletar informações de dispositivos de rede.
    
- Seu uso é **não-intrusivo**, ou seja, **não modifica configurações**, apenas lê dados públicos expostos pelo SNMP.
    

---

## 🎯 Para que serve?

O `snmp-check` é usado principalmente para:

- Descobrir **informações do sistema**: nome do host, localização, contato.
    
- Coletar **interfaces de rede** e IPs configurados.
    
- Verificar **uptime**, **versão de software** e **marca/modelo** do equipamento.
    
- Listar **serviços em execução**.
    
- Obter **informações de roteamento**, **usuários locais**, **processos** e **compartilhamentos** (dependendo do dispositivo).
    

Isso é extremamente útil em **testes de intrusão**, pois:

- Muitos dispositivos mantêm a **comunidade SNMP padrão (`public`)**, que permite acesso a dados sensíveis.
    
- Ajuda o atacante a mapear a rede e identificar alvos vulneráveis.
    

---

## 🛠️ Instalação

Em distribuições baseadas em Debian (como Kali ou Ubuntu):

```bash
sudo apt install snmp-check
```

---

## 📌 Uso básico

```bash
snmp-check -c <comunidade> -v <versão> <IP-alvo>
```

Exemplo:

```bash
snmp-check -c public -v 2c 192.168.0.1
```

---

## 📋 Cheat Sheet (Atalhos e comandos úteis)

|Comando|Descrição|
|---|---|
|`snmp-check <IP>`|Faz scan com comunidade padrão `public` e versão SNMPv1|
|`snmp-check -v 2c -c public <IP>`|Usa SNMPv2c com string `public`|
|`snmp-check -c private <IP>`|Usa comunidade privada (pode revelar info mais sensível se acessível)|
|`snmp-check -p 161 -t 10 -r 3 <IP>`|Ajusta a porta, timeout e tentativas|
|`snmp-check -w <IP>`|Tenta detectar permissões de **escrita** SNMP (potencial crítico de segurança!)|
|`snmp-check -d <IP>`|Desativa enumeração de conexões TCP (menos ruído)|
|`snmp-check -i <IP>`|Exibe apenas interfaces de rede|
|`snmp-check -h`|Ajuda e opções da ferramenta|

---

## 🔍 O que pode ser encontrado (Exemplos de saída)

- **Sistema operacional** e nome de host
    
- **Localização física do dispositivo** (campo `location`)
    
- **Admin responsável** (`contact`)
    
- **Tempo de atividade (uptime)**
    
- **Interfaces de rede** com IPs, MACs, status
    
- **Serviços TCP/UDP escutando**
    
- **Usuários locais**
    
- **Informações do roteador**: rotas, protocolos ativos
    

---

## ⚠️ Riscos de segurança explorados

SNMP mal configurado pode expor:

- **Topologia da rede**
    
- Informações internas sobre servidores e dispositivos
    
- **Contas de usuários**
    
- Serviços abertos
    
- Potencial de **escalonamento de privilégios**, especialmente se a comunidade de **escrita** estiver ativa (`private`)
    

---

## 🧩 Dica complementar: combinar com `onesixtyone` e `snmpwalk`

- **`onesixtyone`**: faz brute force de strings SNMP (comunidades)
    
- **`snmpwalk`**: varredura detalhada de MIBs SNMP
    
- **`snmpenum`**: alternativa automatizada (em Perl)
    

---

Vamos analisar o fluxo de comunicação entre a ferramenta **`snmp-check`** e um dispositivo de rede que possui um agente SNMP configurado. Essa comunicação ocorre por meio de pacotes SNMP encapsulados em datagramas UDP.([GTA UFRJ](https://www.gta.ufrj.br/grad/00_1/joao/snmp.htm))

---

## 📡 Arquitetura da Comunicação SNMP

O SNMP segue uma arquitetura cliente-servidor, onde:([Meu kultura](https://meukultura.com/guia-completo-do-snmp/))

- **Cliente (Gerente SNMP)**: A ferramenta `snmp-check` ou qualquer estação de gerenciamento de rede (NMS).
    
- **Servidor (Agente SNMP)**: O dispositivo de rede gerenciado, como roteadores, switches ou servidores.([itcsecurity.com.br](https://itcsecurity.com.br/pt/aprenda-sobre-snmp-seu-funcionamento-e-todas-as-suas-versoes/))
    

A comunicação é baseada em mensagens SNMP, que são transportadas via **UDP**:

- **Porta 161**: Usada para solicitações e respostas SNMP.
    
- **Porta 162**: Usada para notificações (TRAPs) enviadas pelo agente para o gerente.([GTA UFRJ](https://www.gta.ufrj.br/grad/11_1/snmp/cap3.html), [Site24x7](https://www.site24x7.com/pt/network/what-is-snmp.html))
    

---

## 🧾 Estrutura do Datagram SNMP

Cada mensagem SNMP é encapsulada em um datagrama UDP e segue a estrutura abaixo:

```
+-------------------+-------------------+-------------------+
| Cabeçalho IP      | Cabeçalho UDP     | Mensagem SNMP     |
| (Endereço Origem  | (Porta Origem:    | (Versão, Comunidade, |
| e Destino, Tamanho| 161, Tamanho,     | Tipo de PDU, ID,   |
| etc.)             | Checksum)         | Status, Índice,    |
|                   |                   | Bindings)         |
+-------------------+-------------------+-------------------+
```

### Detalhamento da Mensagem SNMP

A mensagem SNMP, que constitui a PDU (Unidade de Dados de Protocolo), contém os seguintes campos:([GTA UFRJ](https://www.gta.ufrj.br/grad/00_1/joao/snmp.htm))

- **Versão**: Indica a versão do SNMP (1, 2c ou 3).
    
- **Comunidade**: String que atua como uma senha para autenticação simples (ex: "public" ou "private").
    
- **Tipo de PDU**: Define a operação SNMP (GET, GETNEXT, GETBULK, SET, TRAP).
    
- **ID da Solicitação**: Identificador único para correlacionar solicitações e respostas.
    
- **Status de Erro**: Indica se houve erro na operação.
    
- **Índice de Erro**: Aponta o objeto que causou o erro.
    
- **Bindings de Variáveis**: Lista de pares OID (Identificador de Objeto) e valor, representando os dados solicitados ou a serem configurados.([itcsecurity.com.br](https://itcsecurity.com.br/pt/aprenda-sobre-snmp-seu-funcionamento-e-todas-as-suas-versoes/), [Wikipédia, a enciclopédia livre](https://pt.wikipedia.org/wiki/Simple_Network_Management_Protocol))
    

---

## 🔄 Exemplo de Comunicação

1. **Solicitação GET**: O gerente SNMP envia uma solicitação GET para o agente, requisitando informações específicas, como o uptime do dispositivo.
    
2. **Resposta GET-RESPONSE**: O agente processa a solicitação e envia uma resposta contendo os dados requisitados.
    
3. **Notificação TRAP (opcional)**: O agente pode enviar notificações TRAP para o gerente em caso de eventos significativos, como falhas ou mudanças de estado.([Meu kultura](https://meukultura.com/guia-completo-do-snmp/))
    

---

## 🛡️ Considerações de Segurança

É importante destacar que o SNMPv1 e o SNMPv2c utilizam autenticação baseada em comunidade, que é transmitida em texto claro, tornando-os vulneráveis a interceptações. Já o SNMPv3 oferece recursos de segurança aprimorados, incluindo autenticação e criptografia. ([itcsecurity.com.br](https://itcsecurity.com.br/pt/comandos-adicionais-para-testes-snmp/))

---

