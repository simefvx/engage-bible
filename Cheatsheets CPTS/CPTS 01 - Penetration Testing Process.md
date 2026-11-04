---
tema: Pentest / Metodologia
tópico: Processo de Penetration Testing
tags: [htb, cpts, metodologia, processo, roe, escopo, ptes, owasp, mitre]
dificuldade: fundamental
---

# CPTS 01 - Penetration Testing Process — Cheatsheet

> [!info] Escopo deste cartão
> Módulo **teórico**: como um pentest é conduzido ponta a ponta. Não há comando aqui — há **processo, vocabulário, escopo, RoE, metodologias e entregáveis**.
> Use este cartão antes de qualquer engajamento real e ao escrever proposta/relatório.

---

## 🎯 Quando usar

| Situação | O que tirar daqui |
|----------|-------------------|
| Recebeu um pedido de pentest | Seção **Pré-engajamento** + **Perguntas de escopo** |
| Vai assinar/negociar o contrato | Seção **RoE** |
| Não sabe por onde começar no alvo | Seção **Fases do processo** |
| Cliente pediu "só um scan" | Comparativo **Vuln Assessment × Pentest × Red Team** |
| Precisa justificar horas e entregáveis | Seção **Entregáveis** |
| Vai escrever o relatório | [[CPTS 27 - Documentation and Reporting]] |

---

## 🧰 Ferramentas e referências

Este módulo não usa ferramentas de ataque; usa **frameworks** e **documentos de contrato**.

| Recurso | Tipo | Para que serve |
|---------|------|----------------|
| PTES | Metodologia | Fases do pentest, vocabulário padrão de proposta |
| OWASP WSTG | Metodologia | Checklist de teste de aplicação web |
| MITRE ATT&CK | Base de conhecimento | Mapear TTPs do atacante e do que foi testado |
| OWASP Top 10 | Lista de risco | Comunicar risco web para leigos |
| NIST SP 800-115 | Guia | Testes de segurança técnica e avaliação |
| CIS Controls | Controles | Recomendações pós-teste |
| PCI DSS / HIPAA / GDPR / ISO 27001 | Compliance | O que o teste precisa cobrir por lei/auditoria |
| Modelo de escopo + RoE + NDA | Documentos | Formalizar autorização antes de tocar no alvo |

> 💡 Framework não substitui julgamento. Ele dá **cobertura** (não esquecer uma categoria) e **linguagem** (cliente e auditor entendem).

---

## 🧭 Fases do processo

Ciclo iterativo — cada descoberta reabre a fase anterior.

| # | Fase | Objetivo | Saída |
|---|------|----------|-------|
| 1 | Pré-engajamento | Contrato, escopo, RoE, janela, contatos | Autorização assinada |
| 2 | Coleta de informação | Mapear alvo sem explorar | Inventário de hosts/serviços/pessoas |
| 3 | Avaliação de vulnerabilidades | Identificar e **validar** falhas | Lista de falhas com evidência |
| 4 | Exploração | Obter acesso/impacto controlado | Foothold + PoC |
| 5 | Pós-exploração | Medir valor do acesso | Credenciais, dados, privesc |
| 6 | Movimento lateral | Alcançar o objetivo do escopo | Caminho até o alvo final |
| 7 | Proof of Concept | Provar impacto de forma segura | Evidência reproduzível |
| 8 | Pós-engajamento | Documentar e encerrar | Relatório + debrief + limpeza |

```text
Pré-engajamento
   └─> Coleta ──> Vulnerabilidade ──> Exploração ──> Pós-exploração
        ^                                              │
        └────────── nova credencial / novo host ───────┘
                              └─> Relatório é escrito DURANTE tudo
```

---

## 🧪 Tipos de teste

| Eixo | Variantes | O que muda na prática |
|------|-----------|-----------------------|
| Conhecimento do alvo | Black box / Grey box / White box | Começa sem info, com credencial parcial, ou com tudo |
| Origem | Externo / Interno | De fora da internet ou dentro da rede do cliente |
| Alvo | Rede, Web, Mobile, Cloud, API, Wireless, IoT | Ferramentas e escopo diferentes |
| Profundidade | Vulnerability Assessment / Pentest / Red Team | Scan e lista × explorar até o objetivo × simulação adversária completa |
| Anúncio | Anunciado / Não anunciado | Blue team sabe ou não (alerta a SOC) |
| Foco | Evidência (assurance) / Objetivo (goal-based) | Cobertura máxima vs cumprir um cenário específico |

**Assessment × Pentest × Red Team**

- **Vulnerability Assessment** — identificar e listar, validação leve. Menor ruído.
- **Pentest** — explorar de verdade e provar impacto, com regra de parada clara.
- **Red Team** — adversário simulado, foco em objetivo (ex.: exfiltrar dado X), duração longa, highly evasive.

---

## 📝 Pré-engajamento

Tudo que precisa estar fechado **antes** do primeiro pacote na rede.

| Item | Conteúdo |
|------|----------|
| Partes | Quem contrata, quem testa, quem é o contato técnico |
| Escopo | IPs, CIDRs, domínios, apps, contas, redes, e o que está **fora** |
| Janela | Datas, horários, blackout, fuso |
| RoE | O que é permitido e o que **para** o teste |
| Testes | Anunciado? Phishing? DoS? Engenharia social? Social media? |
| Credenciais | Fornece contas? Nível de privilégio? VPN? |
| Ambientes | Produção, staging, homologação — qual pode quebrar |
| Entregáveis | Relatório, debrief, reteste, formato, prazo, idioma |
| Dados | Tratamento de PII/loot, criptografia, retenção, destruição |
| Seguro/legal | Cobertura, NDA, aviso a provedor cloud, notificação a terceiros |
| Emergência | Canal 24h, palavra de parada, procedimento de incidente |

> [!warning] Comunicar alvo fora do escopo
> Host de terceiro apareceu no meio do path? **Pare**, documente e peça autorização por escrito antes de continuar. Escopo é limite legal, não sugestão.

---

## ⚖️ Regras de Engajamento (RoE)

Documento que traduz o contrato em ações permitidas/proibidas no dia a dia.

| Cláusula | Pergunta que responde |
|----------|-----------------------|
| Ativos autorizados | Exatamente quais hosts/apps podem ser tocados |
| Atividades permitidas | Scan, exploit, brute force, phish, upload, pivoting |
| Atividades proibidas | DoS, wipe, persistência, dados reais, sistemas de saúde |
| Janela e limites | Horário, taxa de pacotes, IPs de origem autorizados |
| Tratamento de dados | Onde armazenar, como cifrar, quando destruir |
| Escalonamento | Quem acionar, telefone, tempo de resposta |
| Evidência | O que fotografar/logar em cada achado |
| Parada | Gatilhos de abort (serviço crítico, impacto em pacientes etc.) |
| Reteste | Prazo, escopo, custo e critério de "corrigido" |

---

## 📚 Metodologias e frameworks

| Framework | Fases / Estrutura | Quando escolher |
|-----------|-------------------|-----------------|
| PTES | 7: Pre-engagement, Intelligence Gathering, Threat Modeling, Vuln Analysis, Exploitation, Post-Exploitation, Reporting | Proposta de pentest de infra/adversarial |
| OWASP WSTG | Categorias INFO, CONF, IDNT, ATHN, ATHZ, SESS, INPV, ERRH, CRYP, BUSL, CLNT, APIT | Teste de aplicação web (checklist) |
| OWASP Top 10 | 10 categorias de risco | Comunicar risco a público não técnico |
| MITRE ATT&CK | Táticas: Recon, Resource Dev, Initial Access, Execution, Persistence, PrivEsc, Defense Evasion, Credential Access, Discovery, Lateral Movement, Collection, C2, Exfiltration, Impact | Mapear detecção; nomear TTPs no relatório |
| NIST SP 800-115 | Planejamento → Execução → Pós-execução | Contexto de governança/auditoria |
| OSSTMM | Métricas de segurança operacional | Quando o cliente pede abordagem métrica |

> 💡 Combine: **PTES** para o processo, **WSTG** para a parte web, **ATT&CK** para nomear técnicas. No relatório, cite o framework para dar rastreabilidade.

---

## 🎯 Perguntas de escopo

- Qual o **objetivo de negócio** do teste (comprometimento de dados, compliance, pós-incidente)?
- Quais **IPs/CIDRs/domínios/apps** entram? Qual o inventário oficial?
- Há **exclusões** explícitas (hosts frágeis, sistemas clínicos, produção crítica)?
- Qual o **tipo de teste** (black/grey/white) e o que já é conhecido?
- Podemos usar **credenciais** fornecidas? Até qual privilégio?
- É **anunciado**? A SOC sabe que posso gerar alerta?
- Existe **limite de taxa** ou horário (janela) a respeitar?
- Quem **autoriza** ampliar escopo durante o teste?
- O que acontece se eu **derrubar** um serviço — quem aciono e como?
- Como o **sucesso** será medido (objetivo atingido, cobertura, achados)?
- Formato do **entregável** e prazo? Haverá **reteste**?
- Regras de **dados**: PII, cifra, armazenamento, destruição ao fim?

---

## 📦 Entregáveis

| Entregável | Conteúdo |
|------------|----------|
| Relatório técnico | Achados, evidência, reprodução passo a passo, logs |
| Sumário executivo | Risco de negócio, postura geral, prioridades |
| Classificação de risco | CVSS + contexto de negócio (não só o score) |
| Plano de remediação | Correção, defesa em profundidade, referências |
| Debrief | Reunião de apresentação; Q&A com time técnico |
| Evidência bruta | Capturas, requisições, hashes, timestamps |
| Carta de atestado | Declaração de que o teste ocorreu e o que cobriu |
| Reteste | Prova de correção após remediação |

---

## 🧾 Evidência e documentação

- **Timestamp tudo** — hora local e UTC, sequência de ações.
- **Comando + saída** — o comando exato executado e o resultado colado.
- **Nada de "creio que"** — achado sem evidência é achado contestável.
- **Notas contínuas** — escreva o relatório enquanto testa, não depois.
- **Cuidado com dado real** — PII encontrada é referenciada pelo mínimo necessário.

---

## ⚠️ Pegadinhas

| 🕳️ | Detalhe |
|-----|---------|
| Escopo verbal | "Pode testar aquilo também" por telefone não vale. Formalize por escrito. |
| Escopo de IP errado | CIDR desatualizado → você ataca terceiro. Confirme o inventário. |
| Ignorar a janela | Scan fora do horário autorizado = incidente, mesmo sem impacto. |
| Confundir assessment com pentest | Cliente acha que "rodar scanner" já é pentest; alinhe expectativa. |
| Pular o pós-engajamento | Sem limpeza, artefatos ficam no ambiente — risco e descrédito. |
| Relatório sem contexto de negócio | CVSS alto sem impacto real vira ruído; explique o "e daí?". |
| Prometer "100% de cobertura" | Pentest prova impacto, não garante ausência de falhas. |
| Guardar loot sem regra | Dados de cliente sem cifra/retenção definida = problema legal. |

---

## ✅ Checklist rápido

- [ ] Autorização escrita, escopo e janela confirmados
- [ ] RoE assinado, com gatilhos de parada e contatos de emergência
- [ ] Tipo de teste (black/grey/white) e metodologia definidos
- [ ] Inventário de ativos normalizado e validado pelo cliente
- [ ] Ferramentas e wordlists prontas; rede de ataque (VPN) testada
- [ ] Canal de comunicação e palavra de parada acordados
- [ ] Modelo de relatório e estrutura de evidência prontos **antes** de começar
- [ ] Regras de dados (PII, cifra, retenção) acordadas
- [ ] Pós-teste: relatório entregue, debrief agendado, reteste planejado

---

## 🔗 Relacionado

- [[Fase 01 - Pré-Engajamento]]
- [[Workflow do Engage]]
- [[CPTS 02 - Getting Started]]
- [[CPTS 27 - Documentation and Reporting]]
- [[Fase 12 - Relatório e Encerramento]]
