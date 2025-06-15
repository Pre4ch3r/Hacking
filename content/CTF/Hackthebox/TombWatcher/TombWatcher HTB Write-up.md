---
title: TombWatcher HTB Write-up
created: 2025-06-15 16:33
tags: 
draft: true
---

# Subtítulo da horinha

![[imagem.png | descrição da imagem]]

## Introdução

introdução e resumo aqui

## Reconhecimento

Fase de reconhecimento aqui

## Acesso Inicial

Exploração do alvo (foothold)

## Escalação de Privilégios

Escalada de privilégios até o root

## Conclusão

![[imagem.png | descrição da imagem]]

Palavras finais sobre o que aprendeu e CTA.

```mermaid
flowchart TD
	subgraph acesso inicial
    A(acesso inicial) -->|CVE-2025-xxxx| B(www-data) 
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
