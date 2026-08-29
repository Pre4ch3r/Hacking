---
title: <% await tp.system.prompt("Qual o título?") %>
creation date: <% tp.file.creation_date() %> 
modification date: <% tp.file.last_modified_date() %>
tags: <% tp.file.tags %>
draft: true
---

# Subtítulo da horinha

![descrição da imagem](images/pasta/banner.png)

## Introdução

introdução e resumo aqui

## Reconhecimento

Fase de reconhecimento aqui

## Acesso Inicial

Exploração do alvo (foothold)

## Escalação de Privilégios

Escalada de privilégios até o root

## Conclusão

![[bannerfinal.png | descrição da imagem]]

Palavras finais sobre o que aprendeu.

```mermaid
flowchart TD
	subgraph acesso inicial
    A(acesso inicial) -->|CVE-2026-xxxx| B(www-data) 
    B -->|db.php| C(user1)
    end
    subgraph escalação de privilegios
    C -->|exploit.sh| D(root)
    C -->|portforward| E[web as root]
    E -->|command execution| D(root)
    end
```

Você pode usar links internos: [[outra-nota]]
E inserir imagens: ![[imagem.png | descrição da imagem]]
Ou blocos de código:

```bash
echo "Exemplo de comando"
```