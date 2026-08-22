---
title: CVSS
created: 2025-11-15 09:54
tags: 
unlisted: true
draft: false
---

## CVSS

### O que é o CVSS?

**CVSS (Common Vulnerability Scoring System)** é um padrão que serve para **medir a gravidade de uma vulnerabilidade**.  
Ele transforma várias características técnicas em um **número de 0,0 a 10,0**, indicando quão sério é o problema.

A ideia é simples:

> “Quão fácil é explorar?” + “O que acontece se alguém explorar?”  
> = **Pontuação de risco**

Assim, analistas, empresas e pesquisadores têm uma linguagem comum para priorizar correções.

---

### Os 3 grupos de métricas (de forma simples)

1. **Base** – quão grave é a vulnerabilidade em si  
    (ex.: precisa de login? exige interação do usuário? afeta confidencialidade, integridade, disponibilidade?)
    
2. **Temporal** – muda com o tempo  
    (ex.: existe patch? exploit é conhecido?)
    
3. **Environmental** – depende do ambiente afetado  
    (ex.: o sistema é crítico para a empresa?)
    

A pontuação mostrada publicamente geralmente é a **Base Score**, a mais usada.

---

### Exemplo simples de cálculo

Vamos supor uma vulnerabilidade fictícia:

> **Um site permite que alguém não autenticado acesse dados confidenciais com um link manipulado.**

Como isso viraria CVSS?

### Avaliando os pontos principais

- **Vetor de ataque:** pela internet → ponto positivo para o atacante → aumenta a nota
    
- **Privilegios necessários:** nenhum → aumenta a gravidade
    
- **Interação do usuário:** nenhuma → aumenta
    
- **Impacto na confidencialidade:** alto → aumenta muito
    
- **Integridade / disponibilidade:** não afetadas → pouco impacto aqui

Se colocarmos isso numa calculadora CVSS (como a do FIRST), um conjunto assim costuma gerar algo aproximadamente como:

**CVSS 7.5 (High)**  CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N

Por quê?  
Porque é fácil de explorar, não exige login e expõe dados sensíveis — mas não destrói o sistema nem derruba o serviço.


