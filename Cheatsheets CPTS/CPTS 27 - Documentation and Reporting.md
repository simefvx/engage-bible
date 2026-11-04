---
tema: Reporting
tópico: Documentação e Relatório — notas, findings, CVSS e entrega ao cliente
tags: [htb, cpts, relatorio, documentacao, cvss, obsidian, comunicacao]
dificuldade: easy
---

# 📝 CPTS 27 - Documentation and Reporting

> [!info] Objetivo
> Transformar o caos do engajamento em um **relatório que o cliente consegue usar**: achar o que foi feito, entender o risco e saber exatamente o que corrigir. O relatório é o **produto final** do pentest — sem ele, o trabalho técnico não gera valor. Aqui está a estrutura, o template de finding, a classificação de risco e o processo de qualidade.

---

## 🎯 Quando usar

| Momento | O que fazer |
|---------|-------------|
| **Antes** do engajamento | Escolher ferramenta de notas, montar o esqueleto do relatório, alinhar formato com o cliente |
| **Durante** cada passo | Registrar comandos, saída, timestamp e evidência **na hora** |
| Ao achar um critical | Avisar o cliente imediatamente (não esperar o relatório final) |
| Últimas 48h | Consolidar achados, classificar CVSS, escrever remediações |
| Antes de entregar | Passar pelo checklist de qualidade e revisar por outra pessoa |

> [!tip] Escreva durante, não depois
> Tentar reconstruir "o que eu fiz" uma semana depois gera relatório vago e sem evidência. O relatório se escreve **em paralelo ao engajamento**: cada comando executado vira uma linha de nota.

---

## 🧰 Ferramentas

| Ferramenta | Tipo | Por que usar | Como estruturar |
|-----------|------|--------------|-----------------|
| **Obsidian** | 🟢 local | Markdown puro, wikilinks, grafo de hosts, plugins de tabela | Vault por engajamento: `01-Notas/`, `02-Evidências/`, `03-Relatório/` |
| **CherryTree** | 🟢 local | Árvores hierárquicas, código embutido, sem internet | Nó por host, filho por serviço/vuln |
| **Joplin** | 🟢 sync | Markdown + sincronização criptografada entre máquinas | Notebook por cliente, tags por severidade |
| **Ghostwriter** | 🔵 | Relatórios e templates em equipe, geração de Word | Uso profissional/red team |
| **Notion / OneNote** | 🔵 | Colaboração, checklist | Bom para status com o cliente |
| **Pwndoc / Dradis** | 🔵 | Base de findings reutilizável, export multi-formato | Consultoria com catálogo de vulns |
| **markdown-pdf / pandoc** | 🟢 | Converter Markdown → PDF/HTML/Word | `pandoc relatorio.md -o relatorio.pdf` |

> [!note] Regra prática
> A ferramenta importa menos que o **método**. Markdown + uma pasta de prints resolve a maioria dos engajamentos. O que não pode faltar: comando, saída, data/hora, host e impacto.

---

## 🗂️ Estrutura de notas durante o engajamento

```
Engajamento_<CLIENTE>/
├── 00-Escopo/            # contrato, RoE, escopo, contatos, janelas
├── 01-Recon/             # nmap, dns, subdomínios, tecnologia
├── 02-Hosts/
│   ├── 10.10.10.5.md     # nota por host: portas, serviços, credenciais
│   └── 10.10.10.7.md
├── 03-Credenciais/       # usuário, senha/hash, onde foi achado, onde funciona
├── 04-Payloads/          # scripts, exploits custom, comandos reproduzíveis
├── 05-Evidencias/        # prints, saídas brutas, arquivos capturados
└── 06-Relatorio/         # rascunho, findings, versão final
```

```bash
# 🐧 Salvar saída de comando COM prova de data/hora (vale ouro como evidência)
date -u | tee -a /evidencias/10.10.10.5_root_proof.txt
id | tee -a /evidencias/10.10.10.5_root_proof.txt
hostname; cat /root/proof.txt | tee -a /evidencias/10.10.10.5_root_proof.txt

# 🐧 Nmap sempre com -oA: um arquivo por formato, tudo rastreável
nmap -sC -sV -oA recon/10.10.10.5_full <IP>

# 🐧 Convenção de nome de evidência: <host>_<serviço>_<achado>_<data>.png
mv screenshot.png 10.10.10.5_http_sqli_extract_2026-10-18.png
```

---

## 📐 Estrutura do relatório

| Seção | Público | Conteúdo |
|-------|---------|----------|
| **Capa** | Todos | Cliente, título, versão, data, classificação de confidencialidade |
| **Controle de versão** | Todos | v1.0, autor, revisor, data, mudanças |
| **Sumário executivo** | Gestão / não-técnico | 1–2 páginas, sem jargão: risco geral, o que foi achado, prioridade |
| **Escopo e metodologia** | Todos | Alvos, janelas, limites, padrões usados (`PTES`, `OWASP WSTG`, `MITRE ATT&CK`) |
| **Resumo de achados** | Todos | Tabela: severidade, quantidade, CVSS médio, status |
| **Detalhamento dos achados** | Técnico | Um bloco por vulnerabilidade (template abaixo) |
| **Recomendações / remediação** | Técnico + gestão | Ações priorizadas, curto/médio/longo prazo |
| **Apêndices** | Técnico | Evidências brutas, saídas de ferramenta, mitigações compensatórias |
| **Anexo de reteste** | Todos | O que foi corrigido e o que permanece aberto |

### Sumário executivo — o que dizer

```markdown
## 1. Sumário executivo

Durante o período de <DATA_INICIO> a <DATA_FIM>, conduzimos um teste de intrusão
nos sistemas <ESCOPO>. O objetivo era avaliar a exposição externa e o controle de
acesso interno.

**Resultado geral: risco ALTO.** Um atacante não autenticado conseguia acesso
remoto ao servidor web <HOST> (achado #1) e, a partir dele, obter privilégios de
administrador no domínio (achado #3). Não foi necessário comprometer credenciais
de usuário final.

Foram identificadas **7 vulnerabilidades**: 1 crítica, 2 altas, 3 médias, 1 baixa.
A principal fragilidade é <TEMA_RECORRENTE> (ex.: gestão de patches fora do prazo).

Nenhum dado foi exfiltrado além das cópias necessárias à prova de impacto; todos
os artefatos de teste foram removidos.
```

> [!tip] Regra do C-level
> O sumário executivo responde três perguntas: **o que aconteceu**, **qual o risco ao negócio** e **o que fazer**. Sem screenshots, sem IP, sem `CVE` solto. Se o gestor não entender em 2 minutos, o sumário falhou.

---

## 🧾 Template de finding (achado)

```markdown
### [ID] – Título curto e específico (ex.: Autenticação Ausente na API de Pedidos)

| Campo | Valor |
|-------|-------|
| **Severidade** | Critical / High / Medium / Low / Info |
| **CVSS 3.1** | 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H) |
| **Host / Recurso** | 10.10.10.5:80 – /api/v2/orders |
| **Método** | OWASP WSTG-ATHN-04 |
| **Status** | Confirmado (explorado) / Teórico |
| **Relacionado** | CWE-306, CVE-<ID se aplicável> |

**Descrição**
O endpoint responde a requisições não autenticadas e devolve os pedidos de todos
os clientes, incluindo nome, endereço e últimos 4 dígitos do cartão.

**Reprodução**
1. Acesse `http://<HOST>/api/v2/orders`.
2. Observe o retorno JSON com dados de <N> clientes (evidência #12).
3. Confirme ausência de header `Authorization` e de cookie de sessão no request.

```http
GET /api/v2/orders HTTP/1.1
Host: <HOST>

```

**Impacto**
Qualquer pessoa com acesso à rede (não apenas o ativo interno) lista dados
pessoais de toda a base. Risco de vazamento regulatório (LGPD) e perda de
confiança de titulares.

**Evidência**
[#12] captura do retorno JSON · [#13] saída do `curl -i`.
Timestamp: 2026-10-18 14:32 UTC, host `pwn-attacker`, ausência de autenticação.

**Remediação**
- Aplicar autenticação obrigatória no endpoint e validar autorização por dono do recurso (não só login).
- Retornar somente os campos necessários (mínimo privilégio de dado).
- Adicionar teste automatizado (contrato/CI) que falhe se o endpoint responder 2xx sem token.
Referência: OWASP API Security Top 10 – API1:2023 BOLA.

**Reteste**
Pendente. Aprovado quando `curl` sem token retornar `401`.
```

> [!warning] Finding sem reprodução é achado contestável
> Passos numerados + request/response + evidência datada. O cliente precisa conseguir reproduzir **exatamente** o que você fez para aceitar a remediação e para o reteste.

---

## 📊 CVSS e classificação de risco

**CVSS 3.1 — métricas base (o que vai no relatório)**

| Grupo | Métrica | Valores |
|-------|---------|---------|
| Exploração | **AV** Attack Vector | N (rede) · A (adjacente) · L (local) · P (físico) |
| | **AC** Attack Complexity | L (baixa) · H (alta) |
| | **PR** Privileges Required | N · L · H |
| | **UI** User Interaction | N (nenhuma) · R (requer) |
| Escopo | **S** Scope | U (inalterado) · C (alterado) |
| Impacto | **C / I / A** | H · L · N |

**Faixas de severidade**

| Score | Severidade |
|-------|-----------|
| 9.0 – 10.0 | **Critical** |
| 7.0 – 8.9 | **High** |
| 4.0 – 6.9 | **Medium** |
| 0.1 – 3.9 | **Low** |
| 0.0 | **None / Informational** |

```text
# Exemplo de vetor e leitura:
CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H  = 9.8 Critical
# rede · trivial · sem credencial · sem interação do usuário · impacto total

# Calcule em: https://www.first.org/cvss/calculator/3.1
# CVSS 4.0 existe e adiciona métricas de ameaça/ambiente (use se o cliente exigir).
```

> [!caution] CVSS não é o risco do negócio
> Um `Medium` CVSS em um sistema que guarda dados de saúde pode ser **o achado mais importante do relatório**. Ordene os achados por **risco real** (ativo + dado + exposição) e explique isso ao cliente. Contexto vence número.

---

## 🖼️ Evidência

| Tipo | O que capturar | Cuidado |
|------|----------------|---------|
| Screenshot | `id`/`whoami` + host + data no mesmo print; prova de acesso | Hora visível; sem dados pessoais desnecessários |
| Log bruto | Saída de `nmap`, `-oA`; requests HTTP completos | Salvar o arquivo, não só o print |
| Request/Response | `curl -i`, Burp export, arquivo `.http` | Incluir headers e cookies |
| Vídeo/gif | Cadeia longa de exploração | Útil para crítico |
| Hash/checksum | Prova de artefato capturado | `sha256sum arquivo` |

```bash
# 🐧 Coletar prova de acesso com contexto completo (Linux)
echo "=== $(date -u) host=$(hostname) user=$(id) ===" | tee evidencias/proof.txt
cat /root/proof.txt | tee -a evidencias/proof.txt
```

```powershell
# 🪟 O mesmo no Windows: tudo junto em um arquivo de prova
"=== $(Get-Date -Format o) host=$(hostname) user=$(whoami)" | Out-File C:\temp\proof.txt -Append
net localgroup Administrators | Out-File C:\temp\proof.txt -Append
```

> [!warning] Nunca edite evidência
> Print editado ou log alterado destrói a credibilidade do relatório inteiro. Se precisar ocultar dado sensível, **tarja uniformemente e declare** no apêndice que a tarja existe.

---

## 🛠️ Remediação e escrita orientada à ação

| Ruim (vago) | Bom (acionável) |
|-------------|-----------------|
| "Melhorar a segurança do servidor" | "Aplicar a atualização que corrige o CVE-<ID> no serviço X até 30/11; habilitar MFA no painel" |
| "Senha fraca" | "Aumentar o comprimento mínimo para 14 e bloquear as 10 mil senhas mais comuns; aplicar em <OU>" |
| "Usar HTTPS" | "Emitir certificado válido para <DOMINIO>, aplicar HSTS e redirecionar 80→443" |

- Priorize por **esforço vs impacto**: correções rápidas de alto impacto primeiro (`quick wins`).
- Cada remediação precisa de **responsável implícito** (time de infra, dev, IAM) e **prazo sugerido**.
- Cite padrões quando ajudar: `OWASP` (ASVS/WSTG/API Top 10), `CIS Benchmarks`, `MITRE ATT&CK` (para detecção).
- Separe **mitigação de curto prazo** (compensatória) de **correção definitiva**.

> [!example] Antes / depois de um achado
> **Antes:** "o servidor está desatualizado".
> **Depois:** "o `Apache 2.4.49` em 10.10.10.5 permite leitura arbitrária de arquivos (CVE-2021-41773). Correção: atualizar para 2.4.51+ ou aplicar a mitigação do advisory. Enquanto isso: bloquear travessia `../` no proxy e desabilitar `mod_cgi`."

---

## 🔁 Reteste e ciclo de vida

1. **Entrega v1.0** → cliente recebe, prazo para correção acordado.
2. **Cliente corrige** → informa quais itens e em quais ativos.
3. **Reteste** → reproduza **exatamente** os passos de cada finding corrigido.
4. **Relatório de reteste** → por achado: `Corrigido` / `Parcialmente corrigido` / `Não corrigido` / `Não testável`.
5. **Fechamento** → arquivar evidências e artefatos; destruir dados copiados conforme o contrato.

```text
# Tabela de reteste (apêndice)
| ID | Severidade | Status inicial | Ação do cliente | Resultado do reteste | Data |
|----|-----------|----------------|-----------------|----------------------|------|
| 01 | Critical  | Confirmado     | Atualizou patch | Corrigido            | 2026-11-20 |
| 03 | High      | Confirmado     | Parcial         | Ainda explorável     | 2026-11-20 |
```

> [!tip] "Parcialmente corrigido" é o veredito mais comum
> Patch aplicado em um host do cluster e não no outro; MFA ligado no painel mas não na API. Documente o que **continua aberto** com a mesma evidência de antes.

---

## 💬 Comunicação com o cliente

| Momento | Canal | Conteúdo |
|---------|-------|----------|
| Kickoff | Reunião | Escopo, janelas, contatos de emergência, o que interrompe o teste |
| Status diário/semanal | E-mail/nota | Progresso, bloqueios, achados em alto nível |
| **Crítico imediato** | Telefone/chat definido no RoE | Achado crítico ou exposição de produção — avise **na hora** |
| Debrief final | Reunião | Walkthrough dos achados, perguntas, próximos passos (reteste) |
| Pós-entrega | E-mail | Janela para dúvidas, suporte ao reteste |

> [!warning] Escalada de critical
> Descobriu algo que expõe dados reais de produção, credencial válida de terceiro ou risco de perda de dados? **Pare a exploração, preserve a evidência e comunique** pelo canal definido no RoE. Não continue "só para ver até onde vai".

---

## 🔍 Qualidade antes de entregar

- [ ] Nome do cliente, escopo e datas corretos em todas as páginas
- [ ] Sem `TODO`, `XXX`, placeholders ou notas internas esquecidas
- [ ] Sem código/nome de outro engajamento (cuidado com copiar-coleção entre relatórios)
- [ ] CVSS calculado e **coerente** com a descrição (vetor compatível com o impacto)
- [ ] Cada finding tem descrição, reprodução, impacto, evidência e remediação
- [ ] Todas as evidências numeradas e referenciadas no texto
- [ ] Sumário executivo sem jargão e sem dado sensível desnecessário
- [ ] Achados ordenados por **risco de negócio**, não só por CVSS
- [ ] Remediações acionáveis, com prioridade e prazo
- [ ] Revisor **humano** (outro analista) leu o relatório inteiro
- [ ] Mitigação compensatória indicada para o que não dá para corrigir já
- [ ] Anexos de reteste preparados (mesmo que vazios)
- [ ] Dados sensíveis tratados; confidencialidade marcada
- [ ] Formato de entrega alinhado (PDF assinado / docx / portal)

---

## ✅ Checklist rápido

- [ ] Ferramenta de notas escolhida e vault criado antes do primeiro comando
- [ ] Estrutura de pastas: escopo, recon, hosts, credenciais, evidências, relatório
- [ ] Nmap sempre com `-oA`; toda saída salva em arquivo datado
- [ ] Tabela de credenciais mantida (usuário, segredo, origem, onde funciona)
- [ ] Sumário executivo escrito **antes** dos detalhes (força a visão macro)
- [ ] Template de finding aplicado igual em todos os achados
- [ ] CVSS 3.1 com vetor completo para cada achado
- [ ] Evidência com data/hora/host/usuário
- [ ] Remediação com prioridade, prazo e dono sugerido
- [ ] Checklist de qualidade rodado por um segundo par de olhos
- [ ] Relatório de reteste planejado com o cliente
- [ ] Artefatos de teste removidos e dados copiados destruídos no encerramento

---

## 🔗 Relacionado

- [[Fase 12 - Relatório e Encerramento]] — a fase que executa tudo isto
- [[CPTS 01 - Penetration Testing Process]] — o ciclo em que o relatório se encaixa
- [[CPTS 06 - Vulnerability Assessment]] — da descoberta ao achado classificado
- [[CPTS 28 - Attacking Enterprise Networks]] — engajamento completo que gera este relatório
- [[Fase 01 - Pré-Engajamento]] — escopo, RoE e expectativas acordados antes de tudo
