---
title: Owasp top 10
created: 2026-03-10 21:56
tags:
draft: false
---

# Owasp top 10

![[Pasted image 20260310222147.png]]

O OWASP Top 10 é um documento de referência padrão para desenvolvedores e especialistas em segurança de aplicações web. Ele representa um consenso sobre os riscos de segurança de aplicações mais críticos, baseado em dados de vulnerabilidades reais e severidade de impacto.

## Lista OWASP Top 10 (Edição 2025)

### **A01:2025 - Broken Access Control (Controle de Acesso Quebrado)** 

Refere-se a falhas que permitem que usuários acessem recursos ou executem funções fora de suas permissões. Inclui a violação do princípio do privilégio mínimo, manipulação de IDs para acessar contas de terceiros e bypass de verificações de controle de acesso.
 
### **A02:2025 - Security Misconfiguration (Configuração Incorreta de Segurança)** 

Riscos derivados de configurações de segurança padrão, incompletas ou ad-hoc. Abrange o armazenamento em nuvem aberto, cabeçalhos HTTP mal configurados, mensagens de erro excessivamente detalhadas e serviços desnecessários ativos.

### **A03:2025 - Software Supply Chain Failures (Falhas na Cadeia de Suprimentos de Software)** 

Foca na falta de integridade em componentes de terceiros, bibliotecas e pipelines de CI/CD. O risco reside no uso de dependências comprometidas ou na ausência de verificação de proveniência e assinaturas de código.
 
### **A04:2025 - Cryptographic Failures (Falhas Criptográficas)** 

Exposição de dados sensíveis devido à ausência de criptografia ou ao uso de protocolos e algoritmos fracos. Envolve a proteção inadequada de dados em trânsito (TLS) e em repouso, além do gerenciamento inseguro de chaves.

### **A05:2025 - Injection (Injeção)** 

Ocorre quando dados não confiáveis são enviados a um interpretador como parte de um comando. O interpretador executa comandos não intencionais, permitindo acesso a dados ou execução de código via SQL, NoSQL, OS Command ou LDAP Injection.

### **A06:2025 - Insecure Design (Design Inseguro)** 

Categoria voltada a falhas estruturais e arquiteturais. Diferencia-se de erros de implementação por focar na ausência de controles de segurança na fase de concepção, como a falta de modelagem de ameaças e padrões de design seguro.

### **A07:2025 - Authentication Failures (Falhas de Autenticação)** 

Problemas na confirmação da identidade do usuário. Inclui vulnerabilidades que permitem ataques de preenchimento de credenciais (credential stuffing), força bruta e a ausência de mecanismos robustos de autenticação multifator (MFA).

### **A08:2025 - Software or Data Integrity Failures (Falhas de Integridade de Software ou Dados)** 

Foca em código e infraestrutura que não protegem contra modificações não autorizadas. Inclui a desserialização insegura de objetos e a confiança cega em dados ou atualizações sem verificação de integridade.

### **A09:2025 - Security Logging and Alerting Failures (Falhas de Log e Alerta de Segurança)** 

Incapacidade de registrar, monitorar ou notificar atividades suspeitas. A falta de logs adequados impede a detecção de incidentes em tempo real e compromete a análise forense pós-comprometimento.

### **A10:2025 - Mishandling of Exceptional Conditions (Manuseio Incorreto de Condições Excepcionais)** 

Falhas na gestão de erros e exceções do sistema. O tratamento inadequado pode causar vazamento de informações técnicas (stack traces), estados inconsistentes do sistema ou interrupções de serviço que podem ser exploradas por atacantes.