---
title: <% tp.file.title %>
created: <% tp.file.creation_date() %>
updated: <% tp.file.last_modified_date("dddd Do MMMM YYYY HH:mm:ss") %>
tags: <% tp.file.tags %>
draft: true
---

# <% tp.file.title %>

Conteúdo da sua nota aqui.

## Introduction

introdução e resumo aqui

## Enumeration

Fase de enumeração aqui

## Exploitation

Exploração do alvo (foothold)

## Privilege Escalation

Escalada de privilégios até o root

## Conclusion

Palavras finais sobre o que aprendeu e CTA.

Você pode usar links internos: [[outra-nota]]
E inserir imagens: ![[imagem.png]]
Ou blocos de código:

```bash
echo "Exemplo de comando"
```
