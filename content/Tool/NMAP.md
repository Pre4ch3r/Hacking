---
title: NMAP
created: 2025-05-08 10:17
updated: quinta-feira 8º maio 2025 10:17:14
tags: [nmap, network, tool, concept]
unlisted: true
draft: false
---

# Nmap (Network Mapper)

O **Nmap** é uma das ferramentas mais populares e poderosas de varredura e auditoria de redes. Desenvolvido por **Gordon Lyon (Fyodor)**, é open-source e amplamente utilizado tanto por administradores de sistemas quanto por analistas de segurança.

## O que é?
É uma ferramenta de linha de comando (com interface gráfica opcional chamada **Zenmap**) usada para:
- Varredura de portas
- Detecção de hosts ativos
- Identificação de sistemas operacionais
- Auditorias de segurança

## Principais Funcionalidades
O Nmap serve para:
1. Descobrir **hosts ativos** em uma rede.
2. Verificar quais **portas estão abertas** e quais **serviços** estão sendo executados.
3. Identificar o **sistema operacional** e o **uptime** do host.
4. Realizar detecção de **firewalls**, **IDS** (Intrusion Detection Systems) e **IPS** (Intrusion Prevention Systems).
5. Auxiliar na **evasão de mecanismos de segurança** para testes controlados.

## Nmap Scripting Engine (NSE)
A opção `--script` ativa o **Nmap Scripting Engine**, que utiliza scripts escritos em **Lua** para varreduras avançadas.
- **Objetivos**: Detecção de vulnerabilidades, autenticação em serviços e enumeração de usuários.
- **Categorias comuns**:
  - `auth`: scripts de autenticação.
  - `vuln`: detecção de vulnerabilidades.
  - `default`: executados com a opção `-sC`.
  - `safe`: seguros para uso regular.
  - `intrusive`: podem causar impacto.

## Sintaxe Básica
É possível usar categorias ou expressões personalizadas.
::search[Nmap scripting engine syntax]{type=web}

> [!WARNING] Ética e Legalidade
> Usar o Nmap para escanear redes ou sistemas **sem permissão é ilegal** e pode resultar em sérias consequências legais. Ferramentas poderosas exigem responsabilidade.

## Ver também
- Exemplos práticos de varredura e comandos: [[Nmap-Cheatsheet]]


