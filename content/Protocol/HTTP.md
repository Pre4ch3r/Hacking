---
title: HTTP
created: 2025-11-15 09:34
tags: 
draft: false
---

# HTTP

## O que é o protocolo HTTP?

HTTP significa **HyperText Transfer Protocol**. Em palavras de gente normal: é o conjunto de regras que permite que seu navegador (Chrome, Firefox, etc.) converse com servidores na internet.  
É ele que define como pedir uma página, como o servidor deve responder, o que fazer quando algo dá errado, etc.

É tipo o “combinado” universal para que tudo funcione.

---

### Como o HTTP funciona na prática?

Quando você acessa um site:

1. O navegador envia uma **requisição HTTP** (um pedido).
    
2. O servidor recebe o pedido.
    
3. Ele responde com uma **resposta HTTP**: pode ser a página, uma imagem, um erro, ou qualquer outro recurso.
    

Parece simples, e é! Justamente por isso o HTTP se tornou tão popular.

### Métodos (ou verbos) – o que o navegador quer fazer

O navegador usa “verbos” pra dizer ao servidor o que ele quer. Os principais:

- **GET**: pedir um recurso (ex.: mostrar a página).
    
- **POST**: enviar dados (ex.: login, formulários).
    
- **PUT / PATCH**: atualizar algo.
    
- **DELETE**: remover algo.
    

Pense nisso como “ações” que seu navegador solicita.

### Status Codes – a resposta do servidor

Sabe o famoso _404 Not Found_? Então, isso é um **status code**.  
Alguns exemplos:

- **200 OK** – tudo certo.
    
- **301/302** – você foi redirecionado.
    
- **404** – o recurso não existe.
    
- **500** – erro do servidor.
    

Eles resumem o que aconteceu sem precisar explicar em texto.

### HTTP vs. HTTPS

Essa é a parte que mais chama atenção na cibersegurança.

- **HTTP é inseguro**: os dados trafegam _em texto puro_.  
    Isso significa que, em teoria, alguém pode interceptar e ler o que está sendo enviado.
    
- **HTTPS é a versão segura**: usa criptografia (TLS) para proteger as informações.  
    Por isso, hoje praticamente todos os sites usam HTTPS.
    

Se tem um cadeado no navegador, é o HTTPS trabalhando.

---

## HTTPS: o HTTP com armadura

HTTPS nada mais é que **HTTP + TLS (Transport Layer Security)**.  
O TLS é responsável por três proteções importantes:

### 1. **Criptografia**

Os dados enviados entre você e o servidor são embaralhados.  
Se alguém interceptar, só vai ver um monte de bytes sem sentido.

### 2. **Autenticação**

Seu navegador verifica se o servidor é realmente quem ele diz ser, através de **certificados digitais**.  
Isso evita que alguém tente se passar por um site legítimo.

### 3. **Integridade**

Garante que ninguém alterou a mensagem no caminho.

Ou seja: HTTPS não muda a forma como o HTTP funciona — ele só protege o transporte das mensagens.

---

### Cabeçalhos HTTP: o “meta” da comunicação

Se o corpo da requisição é o conteúdo, os **cabeçalhos (headers)** são as regras de comportamento.

Alguns dos que mais aparecem (e que fazem diferença na segurança):

- **User-Agent** → Identifica o navegador.
    
- **Cookie** → Traz dados de sessão (por isso é tão sensível).
    
- **Content-Type** → Diz qual é o tipo do conteúdo (HTML? JSON? Imagem?).
    
- **Host** → Informa qual domínio você quer acessar (importante em servidores com vários sites).
    

E no lado da resposta:

- **Set-Cookie** → O servidor cria ou atualiza cookies.
    
- **Server** → Diz qual software está rodando (às vezes demais…).
    
- **Content-Security-Policy (CSP)** → Ajuda a impedir ataques de injeção.
    
- **Strict-Transport-Security (HSTS)** → Diz ao navegador: “Acesse este site sempre via HTTPS”.
    

Esses últimos dois são ouro pra segurança.

---

## Como analisar HTTP 

O jeito mais comum de olhar requisições e respostas é usando ferramentas do próprio navegador:

### DevTools (F12) > Aba “Network”

Ali dá para ver:

- cada requisição feita pela página,
    
- os métodos (GET, POST…),
    
- os cabeçalhos,
    
- os status codes,
    
- o tempo de carregamento,
    
- e até o conteúdo retornado (quando é seguro exibir).
    

É como ter “raios X” da comunicação web.

### Proxies de inspeção (como Burp Suite Community)

Ferramentas usadas para **testes de segurança autorizados**.  
A versão gratuita mostra as requisições, como o navegador realmente as envia, e permite estudar a estrutura do HTTP com mais clareza.

**Importante:** usar essas ferramentas com sites alheios sem permissão é ilegal.  
Mas para aprender, dá pra rodar um ambiente local (tipo uma aplicação de teste) e explorar sem riscos.

---

Aqui vai mais uma “entrada de diário” no estilo estudante de cibersegurança — clara, leve e interessante, sem entrar em nada perigoso.

---

## Cookies — o pequeno arquivo que carrega grande responsabilidade

### O que são cookies?

Cookies são **pequenos pedaços de informação** que o site guarda no seu navegador.  
Eles existem para lembrar coisas que o servidor não consegue lembrar sozinho.

Lembre-se: **HTTP é “stateless”** — ou seja, cada requisição é independente.  
Sem cookies, toda página nova faria o site “esquecer” quem você é.

---

## Para que eles servem?

Aqui vão os usos mais comuns:

### 1. Manter você logado

Quando você faz login, o servidor cria um **identificador de sessão** e coloca esse ID em um cookie.  
Assim, na próxima página, ele reconhece: “Ah, é o mesmo usuário”.

### 2. Guardar preferências

Idioma, tema claro/escuro, carrinho de compras… tudo pode ficar salvo em cookies.

### 3. Analytics e personalização

Sites usam cookies para entender comportamento geral (ex.: páginas mais populares).  
Mas aqui também entra o uso controverso: cookies para anúncios direcionados — por isso toda aquela barra de “aceite os cookies”.

---

## Cookies importantes na segurança

Alguns atributos fazem toda a diferença:

### HttpOnly

Impede que scripts da página leiam o cookie.  
Protege contra certos ataques que tentam “roubar” cookies do navegador.

### Secure

Só permite enviar o cookie via **HTTPS**.  
Nada de trafegar dados sensíveis em canais desprotegidos.

### SameSite

Controla quando o navegador envia cookies entre domínios diferentes.  
Ajuda a mitigar ataques baseados em requisições automáticas (como CSRF).

### Expiration

Define quanto tempo o cookie vai durar.  
Alguns expiram quando você fecha o navegador, outros ficam por semanas ou meses.

---

### Riscos 

Cookies não são perigosos por si só — **o problema é o que eles representam**.

Por exemplo: se um cookie guarda sua sessão, então quem pegar esse cookie “vira você” para o site.  
Isso explica por que os atributos Secure e HttpOnly são tão importantes.

Mas vale reforçar: _não existe risco só por você “ter cookies”_. É o mau uso deles que abre brechas.

---

### Como visualizar cookies?

Sem mistério: no navegador mesmo.

- Chrome/Edge/Brave: **F12 → Application → Cookies**
    
- Firefox: **F12 → Storage → Cookies**
    

Dá para ver nome, valor, flags de segurança (Secure, HttpOnly…), domínio e data de expiração.

É muito útil para quem está estudando como sites mantêm sessões.
