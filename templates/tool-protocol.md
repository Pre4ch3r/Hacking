---
title: <% await tp.system.prompt("Qual o título?") %>
creation date: <% tp.file.creation_date() %> 
modification date: <% tp.file.last_modified_date() %>
tags: <% tp.file.tags %>
unlisted: true
draft: true
---

# <% tp.file.title %>