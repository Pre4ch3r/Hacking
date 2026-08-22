---
title: UDP
created: 2025-05-08 01:23
tags: [udp, protocol, network]
draft: false
---

# UDP

---

### 📦 O que é **UDP**?

**UDP (User Datagram Protocol)** é um **protocolo de comunicação** usado para enviar dados entre computadores em redes, como a internet ou redes locais. Ele faz parte da **camada de transporte** do modelo OSI, assim como o TCP.

---

### ⚙️ Características principais do UDP:

1. **Sem conexão (connectionless):**  
    Não estabelece uma conexão fixa antes de enviar dados — simplesmente envia os pacotes.
    
2. **Sem verificação de entrega (não confiável):**  
    Ele **não garante** que os dados vão chegar, nem em ordem, nem sem duplicação. Se algo se perder, o UDP não tenta reenviar.
    
3. **Leve e rápido:**  
    Por ser simples e ter menos verificações, o UDP é **mais rápido** que o TCP — ideal para tempo real ou transmissão contínua.
    
4. **Usa pacotes chamados "datagramas":**  
    Cada mensagem enviada via UDP é um **datagrama independente**, sem relação com o anterior ou o seguinte.
    

---

### 📡 Exemplos de uso do UDP:

- **Mosh** (conexão remota como SSH)
    
- **DNS** (resolução de nomes)
    
- **VoIP** (voz pela internet)
    
- **Jogos online** (baixa latência)
    
- **Streaming ao vivo**
    

---

### 🆚 Comparação rápida com TCP:

|Característica|TCP|UDP|
|---|---|---|
|Confiável?|Sim (confirma entrega)|Não|
|Ordem garantida?|Sim|Não|
|Conexão necessária?|Sim (3-way handshake)|Não|
|Velocidade|Mais lento|Mais rápido|
|Casos de uso|Web, email, arquivos|Streaming, jogos, VoIP|

---

Resumindo:

> **UDP é rápido, leve e sem garantias. Ideal para aplicações onde a velocidade é mais importante que a confiabilidade.**

