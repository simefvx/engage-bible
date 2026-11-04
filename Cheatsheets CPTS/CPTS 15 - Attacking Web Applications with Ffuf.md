---
tema: Fuzzing Web
tópico: Enumeração e brute force de conteúdo, vhosts, parâmetros e valores com ffuf
tags: [htb, cpts, ffuf, fuzzing, wordlist, vhost, enumeration, web]
dificuldade: easy
---

# CPTS 15 — Attacking Web Applications with Ffuf

> [!info] Objetivo
> Usar o `ffuf` para responder perguntas concretas: quais diretórios e arquivos existem, quais vhosts/subdomínios o servidor atende, quais parâmetros a aplicação aceita e quais valores disparam comportamento diferente. O segredo é **filtrar o ruído** (respostas default) antes de escalar o ataque.

## 🎯 Quando usar

- Precisa enumerar conteúdo web e o `gobuster`/`dirb` não dá controle suficiente de matchers.
- Suspeita de virtual hosts (mesmo IP, aplicações diferentes por `Host:`).
- Precisa descobrir parâmetros escondidos (inclusive POST) ou fuzzar valores de um parâmetro já conhecido.
- Brute force de login/API em que você controla a requisição inteira (headers, cookies, JSON).

## 🧰 Ferramentas

| Ferramenta | Diferença do ffuf |
|---|---|
| `ffuf` | Matchers/filters granulares, recursão, multipalavra, alta performance |
| `gobuster dir/vhost/dns` | Mais simples, bom para dir e vhost rápidos |
| `feroxbuster` | Recursivo por padrão, ótimo para árvores grandes |
| `dirsearch` | Boa lista de extensões default, interface amigável |
| `wfuzz` | Fuzzing por parts (headers, cookies) com filtros `--hh/--hc` |
| `ffuf` + Burp (`-x`) | Registrar tudo o que o fuzzer fez no histórico do proxy |

## 🔤 Sintaxe base

```bash
ffuf -u <URL>/FUZZ -w <WORDLIST> [opções]

# A palavra-chave é FUZZ e pode aparecer em qualquer lugar: path, query, header, body.
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<IP>/FUZZ
```

| Flag | O que faz | Quando usar |
|---|---|---|
| `-u` | URL alvo (contém `FUZZ`) | Sempre |
| `-w` | Wordlist (`arquivo:CUSTOM`) | Sempre; `:CUSTOM` para várias listas |
| `-X` | Método HTTP (`POST`, `PUT`...) | Brute force de formulário/API |
| `-d` | Corpo da requisição | POST com `FUZZ` no body |
| `-H` | Header extra (repetível) | Host virtual, `Content-Type`, auth |
| `-b` | Cookie | Sessão/autenticação |
| `-r` | Seguir redirects | Apps que redirecionam para `/login` |
| `-k` | Ignora TLS inválido | Labs com certificado self-signed |
| `-c` | Saída colorida | Interativo |
| `-s` | Silencioso | Combinar com `-o` para automação |
| `-v` | Mostra URL completa de cada hit | Debugar matcher muito permissivo |
| `-t` | Threads (default 40) | Aumentar com cuidado, reduzir se derrubar |
| `-rate` | Requisições por segundo (limite global) | Alvo frágil ou WAF |
| `-p` | Delay entre requisições (`0.1` ou `0.1-2.0`) | Evitar rate limit |
| `-timeout` | Timeout por requisição (segundos) | Servidor lento |
| `-x` | Proxy HTTP/SOCKS | Burp/ZAP no meio |
| `-replay-proxy` | Só os MATCHES vão para o proxy | Investigar hits sem poluir o log |
| `-e` | Extensões (`.php,.txt`) | Brute force de arquivos |
| `-recursion` | Recursão em diretórios achados | Árvore profunda |
| `-recursion-depth` | Profundidade máxima | Controlar explosão combinatória |
| `-ac` | Autocalibração (descobre filtro default) | Servidor com 404 customizado |
| `-o` / `-of` | Arquivo e formato de saída (`json`,`csv`,`html`,`md`,`all`) | Evidência e pós-processamento |

## 📚 Wordlists

```bash
# Conteúdo web
/usr/share/seclists/Discovery/Web-Content/common.txt                 # 4.7k linhas, rápido
/usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
/usr/share/seclists/Discovery/Web-Content/api/objects.txt
/usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt   # parâmetros

# DNS/vhost
/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt
/usr/share/seclists/Discovery/DNS/namelist.txt

# Senhas
/usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt
```

**Regra de ouro:** comece pequeno (`common.txt`) para mapear a estrutura, depois use `raft-medium` nas pastas que responderam.

## 📁 Dir/File fuzzing

```bash
# Descoberta de diretórios
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt -u http://<IP>/FUZZ \
     -mc 200,204,301,302,307,401,403 -c

# Com extensões e recursão controlada
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt -u http://<IP>/FUZZ \
     -e .php,.html,.txt,.bak -recursion -recursion-depth 2 -t 60

# Auto-calibração quando o 404 devolve 200 com página "Not Found"
ffuf -w list.txt -u http://<IP>/FUZZ -ac -c

# Salvando resultado em JSON para revisar depois
ffuf -w list.txt -u http://<IP>/FUZZ -s -o ffuf_dirs.json -of json
```

## 🏷️ Vhost e subdomínio

```bash
# Vhost: mesma URL, Host header fuzzado. O alvo é o IP para não depender de DNS
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u http://<IP> -H 'Host: FUZZ.<DOMINIO>' -fs 4242 -c
# -fs: filtra o tamanho da resposta DEFAULT (o vhost inexistente). Pegue esse número
#     rodando uma vez sem filtro e olhando o tamanho repetido.

# Subdomínio por DNS (requer resolver configurado)
ffuf -w subdomains.txt -u https://FUZZ.<DOMINIO>/ -mc 200,301,302,401,403 -c

# Combinando saída com /etc/hosts para navegar depois
echo "<IP> app.<DOMINIO>" | sudo tee -a /etc/hosts
```

**Quando usar:** aplicação "principal" bloqueada, mas o mesmo servidor atende um painel interno em outro vhost não listado no DNS público.

## 🔎 Parâmetros (GET e POST)

```bash
# Descobrir NOME de parâmetro GET: compare com o tamanho da resposta SEM parâmetro
curl -s http://<IP>/index.php | wc -c        # ex.: 2048
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -u 'http://<IP>/index.php?FUZZ=test' -fs 2048 -c

# Descobrir parâmetro em POST
ffuf -w burp-parameter-names.txt -u http://<IP>/admin/index.php -X POST \
     -d 'FUZZ=test' -H 'Content-Type: application/x-www-form-urlencoded' -fs 1200 -c

# Fuzzar VALOR de um parâmetro já descoberto (duas wordlists)
ffuf -w params.txt:FUZZ -w values.txt:VAL \
     -u 'http://<IP>/index.php?FUZZ=VAL' -fs 2048 -c

# Dois keywords distintos na mesma requisição (nomes casados por índice com Pitchfork-like)
ffuf -w params.txt:FIELD -w values.txt:VALUE \
     -u 'http://<IP>/index.php?FIELD=VALUE' -fs 2048 -c
```

## 🚪 POST, headers, cookies e JSON

```bash
# Formulário de login: filtre o tamanho/CÓDIGO da resposta de falha
ffuf -w /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt \
     -u http://<IP>/login -X POST \
     -d 'username=admin&password=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' \
     -fc 401 -fs 1200 -c

# API JSON com token de sessão
ffuf -w resources.txt -u https://<IP>/api/FUZZ \
     -H 'Authorization: Bearer <TOKEN>' -H 'Accept: application/json' -mc all -fr 'not found'

# Trocar cookie/User-Agent por requisição
ffuf -w sessions.txt -u http://<IP>/dashboard -b 'PHPSESSID=FUZZ' -mc 200 -fs 340
ffuf -w uas.txt -u http://<IP>/ -H 'User-Agent: FUZZ' -mc 200
```

## 🎯 Matchers e Filters

| Matcher | Filtra por | Filter | Exclui por |
|---|---|---|---|
| `-mc` | status code (ex.: `-mc 200,301`, `-mc all`) | `-fc` | status code |
| `-ms` | tamanho da resposta | `-fs` | tamanho |
| `-ml` | número de linhas | `-fl` | linhas |
| `-mw` | número de palavras | `-fw` | palavras |
| `-mr` | regex no corpo | `-fr` | regex no corpo |
| `-mt` | tempo de resposta (ex.: `-mt >2000`) | `-ft` | tempo |

```bash
# Fluxo recomendado: rode UMA vez sem filtro e observe o padrão dominante
ffuf -w common.txt -u http://<IP>/FUZZ -c | sort -k4 -n | head -20
# Depois filtre exatamente esse tamanho/linhas/palavras:
ffuf -w common.txt -u http://<IP>/FUZZ -fs 4242 -fw 100 -fl 20 -mc all -c

# ERRO CLÁSSICO: filtrar o tamanho errado e perder os hits verdadeiros.
# Confirme sempre com um hit conhecido (ex.: /index.php deve aparecer).
```

**Quando usar cada um:** `-fc` para 404/403 barulhentos, `-fs`/`-fw`/`-fl` quando todo 404 devolve 200 com página genérica (mais preciso que `-fc`), `-mr`/`-fr` quando o sinal está no conteúdo (ex.: mensagem de erro SQL).

## 🔨 Receitas prontas

```bash
# 1) Enumeração completa e rápida de uma app
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
     -u http://<IP>/FUZZ -e .php,.html,.txt -mc 200,204,301,302,401,403 \
     -ac -t 40 -c -o dirs.json -of json

# 2) Vhost discovery com filtro de tamanho
ffuf -w subdomains-top1million-5000.txt -u http://<IP> -H 'Host: FUZZ.<DOMINIO>' -fs 4242 -c

# 3) Parameter discovery + fuzzing de valor
ffuf -w burp-parameter-names.txt -u 'http://<IP>/index.php?FUZZ=1' -fs 2048 -c   # nomes
ffuf -w values.txt -u 'http://<IP>/index.php?cmd=FUZZ' -fs 2048 -c               # valores

# 4) Brute force de login sem travar o alvo
ffuf -w pass10k.txt -u http://<IP>/login -X POST -d 'user=admin&pass=FUZZ' \
     -fc 401 -fs 62 -rate 20 -t 10 -c

# 5) Fuzz através do Burp para registrar tudo
ffuf -w common.txt -u http://<IP>/FUZZ -x http://127.0.0.1:8080 -replay-proxy http://127.0.0.1:8080 -c

# 6) Reaproveitar a requisição exata do Burp (raw request em arquivo)
#    Salve a requisição (Ctrl+R → botão direito → Copy to file) e rode:
ffuf -request req.txt -request-proto https -w params.txt:FUZZ -mc all -fs 1000 -c

# 7) SILENCIOSO para pipeline: só as URLs encontradas
ffuf -w common.txt -u http://<IP>/FUZZ -mc 200,301 -s | tee found.txt

# 8) Recursão agressiva em uma pasta promissora
ffuf -w raft-medium-files.txt -u http://<IP>/admin/FUZZ -e .php -recursion -recursion-depth 3 -t 50
```

## ⚠️ Pegadinhas

- **Sem filtro, `/FUZZ` retorna milhares de hits falsos.** Sempre calibre antes de escalar.
- **`-fs` com valor errado esconde o alvo**: valide com um diretório que você sabe que existe.
- **`-t 40` pode derrubar o alvo ou acionar WAF**: reduza com `-rate` e `-p`.
- **Recursão sem `-recursion-depth`** explode em tempo e requisições.
- **`-e` é lista de extensões, sem curinga**: `-e .php,.bak`, não `-e *.php`.
- **Extensão errada gera 200 falso** em servidores que fazem *trailing path* (`/admin.php/`).
- **`-mc all`** é útil para achar 403/500 escondidos, mas exige filtro forte de tamanho.
- **Vhost precisa de `-u` com IP**: usar o domínio resolve pelo DNS local e você mede o vhost errado.
- **HTTPS com certificado self-signed** falha silenciosamente sem `-k`.
- **Palavras com `%`/espaços** na wordlist quebram a URL — codifique ou use `-H`/`-d`.
- **Kali tem várias wordlists**: confirme o caminho do arquivo antes de rodar (`ls`), ou use `-w` errado e receba 0 hits.
- **`-o` sobrescreve** o arquivo anterior; versionar nomes por alvo.

## ✅ Checklist rápido

- [ ] Alvo responde e a URL base está correta (`curl -I`)
- [ ] Rodada de calibração: descobrir tamanho/linhas padrão do "não existe"
- [ ] Filtro aplicado (`-fs`/`-fw`/`-fl`/`-fc`) e validado com hit conhecido
- [ ] Wordlist adequada ao estágio (common → raft/directory-list)
- [ ] `-e` definido se o servidor usa extensões
- [ ] Vhosts testados quando o alvo é IP + domínio
- [ ] Parâmetros GET e POST descobertos e depois fuzzados por valor
- [ ] Rate/threads ajustados (não derrubar produção)
- [ ] Recursão ligada só onde faz sentido, com profundidade
- [ ] Resultado salvo (`-o ... -of json/html`) como evidência
- [ ] Hits reais reproduzidos manualmente no Burp/curl

## 🔗 Relacionado

- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 05 - Information Gathering - Web Edition]]
- [[CPTS 16 - Login Brute Forcing]]
- [[Fase 05 - Enumeração Web]]
- [[CPTS 24 - Attacking Common Applications]]
