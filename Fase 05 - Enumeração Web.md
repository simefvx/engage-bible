---
tema: Pentest
tópico: Fase 05 — Enumeração Web
tags: [htb, cpts, fase, web, enumeracao, ffuf, gobuster, feroxbuster, burp]
dificuldade: fundamental
---

# 🌐 Fase 05 — Enumeração Web

> [!info] Objetivo da fase
> Transformar "tem um site na porta 80" em um **mapa completo da aplicação**: tecnologia, diretórios, arquivos, hosts virtuais, parâmetros, formulários, sessões e endpoints autenticados. Sem isso, a Fase 06 vira chute. A regra é: **enumere até doer**, e depois enumere mais uma vez com o contexto que você aprendeu.

## 🎯 Quando entrar

- Nmap mostrou HTTP/HTTPS (80, 443, 8080, 8000, 8443, 5000, 3000, 9000, ou qualquer porta que responda a `curl -I`).
- Você achou um vhost, um subdomínio novo, ou uma API documentada no recon externo.
- Você obteve credencial válida (ou de baixo privilégio) e precisa mapear a área autenticada — **a enumeração pós-login é outra aplicação**.
- Sempre que uma fase posterior devolver uma URL nova, um cookie novo ou um erro novo: volte para cá.

## 📥 Pré-requisitos e inputs

| Input | De onde vem |
|-------|-------------|
| IP/host + portas HTTP | Fase 03 (nmap) |
| Domínios e vhosts prováveis | Fase 02 (OSINT), `Host:` nos redirects |
| Tecnologia observada | Headers, favicon hash, `whatweb` |
| Wordlists | `/usr/share/seclists/Discovery/Web-Content/` |
| Proxy configurado | Burp/ZAP ouvindo em `127.0.0.1:8080` |
| Credenciais de teste | Escopo grey-box ou contas criadas por você |
| Escopo/janela | Fase 01 — fuzzing agressivo pode derrubar app frágil |

## 🪜 Passo a passo

### 1. Sempre resolva o nome antes de tudo

Um IP pode servir vários sites. Descubra o vhost canônico antes de fuzzar, senão todos os resultados serão do site default.

```bash
# Pede a raiz e mostra só os headers — revela redirect, servidor, cookies, tecnologias
curl -sI http://<IP>/

# Segue o redirect e mostra o corpo final — descobre o Host correto
curl -sL http://<IP>/ | head -40
```

Se aparecer um domínio, adicione em `/etc/hosts`:

```bash
echo "<IP>  <DOMINIO>" | sudo tee -a /etc/hosts
```

### 2. Fingerprint da aplicação

Descubra o que roda para escolher wordlists e extensões certas. Fuzzar `.php` em app Node é desperdício.

```bash
# Identifica CMS, framework, servidor, JS libs, meta generator
whatweb -a 3 http://<DOMINIO>/

# Scripts NSE de HTTP: título, headers, métodos permitidos, arquivos comuns
nmap -p 80,443 --script http-title,http-headers,http-methods,http-enum <IP>

# Headers revelam proxy reverso, tecnologia e às vezes versão exata
curl -sI http://<DOMINIO>/ | grep -iE 'server|x-powered-by|set-cookie|x-aspnet|via|x-'

# Favicon hash: o mesmo hash aparece no Shodan mesmo atrás de CDN
curl -s http://<DOMINIO>/favicon.ico | mmh3sum.py
```

### 3. Caminhos "de graça" — os que todo mundo esquece

Antes de fuzzar, leia os arquivos que a aplicação **entrega** de propósito ou por descuido.

```bash
curl -s http://<DOMINIO>/robots.txt     # caminhos que o dono não quer indexar = ouro
curl -s http://<DOMINIO>/sitemap.xml    # árvore de URLs oficiais
curl -s http://<DOMINIO>/.well-known/security.txt
curl -s http://<DOMINIO>/crossdomain.xml
```

Teste manualmente os clássicos de vazamento:

```bash
for f in .git/HEAD .env .svn/entries .DS_Store backup.zip backup.tar.gz \
         config.php.bak web.config appsettings.json package.json \
         phpinfo.php server-status api/ swagger.json openapi.json; do
  code=$(curl -s -o /dev/null -w '%{http_code}' "http://<DOMINIO>/$f")
  [ "$code" != "404" ] && echo "$code  $f"
done
```

### 4. Fuzzing de diretórios e arquivos

O coração da fase. Sempre fuzz **diretórios** (recursivo) e **arquivos** (com extensões).

```bash
# gobuster: rápido, simples, bom para um passe inicial
gobuster dir -u http://<DOMINIO>/ \
  -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt \
  -x php,html,txt,bak,zip -t 40 -o gobuster.txt

# feroxbuster: recursivo por padrão, filtra falsos positivos automaticamente
feroxbuster -u http://<DOMINIO>/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,bak,txt -d 3 --auto-tune -o feroxbuster.txt

# ffuf: o mais flexível — filtra por tamanho/status/palavras
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
     -u http://<DOMINIO>/FUZZ -mc all -fc 404 -t 50 -o ffuf.json -of json

# Fuzz de extensões num arquivo conhecido
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt \
     -u http://<DOMINIO>/indexFUZZ

# Backup de arquivo conhecido: troca extensão e adiciona sufixos
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-small-files.txt \
     -u http://<DOMINIO>/configFUZZ -mc all -fs 0
```

> [!tip] Por que filtrar por tamanho (`-fs`) e não só status
> Muitas apps devolvem `200` com página de erro genérica. Se 100 resultados têm 1234 bytes e um tem 8890, o de 8890 é o real. Sempre rode um passe com `-mc all` primeiro para **medir**, depois filtre.

### 5. Virtual hosts (vhost fuzzing)

Um mesmo IP serve vários sites pelo header `Host`. O site interessante quase nunca é o default.

```bash
# Descobre subdomínios/vhosts — filtra pelo tamanho da resposta default
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -u http://<DOMINIO>/ -H "Host: FUZZ.<DOMINIO>" -fs <TAMANHO_DEFAULT>

# gobuster também faz vhost
gobuster vhost -u http://<DOMINIO>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain
```

Rode primeiro sem filtro, veja qual é o tamanho da resposta "não encontrado", e use esse valor em `-fs`. Achei um vhost novo? **Volte ao passo 1 para ele.**

### 6. Descoberta de parâmetros

Endpoints aceitam parâmetros que não aparecem em lugar nenhum. Um `?id=` escondido é uma IDOR/SQLi esperando.

```bash
# arjun: descobre parâmetros GET/POST/JSON por análise de resposta
arjun -u "http://<DOMINIO>/api/user" -m GET
arjun -u "http://<DOMINIO>/login" -m POST

# ffuf faz o mesmo: fuzz no NOME do parâmetro e observe mudança de tamanho
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -u "http://<DOMINIO>/page?FUZZ=test" -fs <TAMANHO_BASE>

# Parâmetros em corpo POST de formulário
ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
     -X POST -d "FUZZ=test" -H "Content-Type: application/x-www-form-urlencoded" \
     -u http://<DOMINIO>/search -fs <TAMANHO_BASE>
```

### 7. Crawling e coleta de URLs

Crawler encontra o que a wordlist não tem: links parametrizados, formulários, arquivos JS com rotas.

```bash
# katana: crawler moderno, executa JS e segue formulários
katana -u http://<DOMINIO>/ -jc -d 3 -aff -o katana.txt

# hakrawler: pipeline simples, ótimo com outras ferramentas
cat urls.txt | hakrawler -d 3 -u

# URLs históricas (Wayback, Common Crawl, AlienVault)
echo <DOMINIO> | gau --threads 5 | tee gau.txt
echo <DOMINIO> | waybackurls | tee wayback.txt

# Extrai segredos de arquivos JS
cat katana.txt | grep -E '\.js$' | sort -u | while read u; do
  curl -s "$u" | grep -Ei 'api[_-]?key|token|secret|/api/v[0-9]|password'
done
```

### 8. Burp / ZAP — onde a análise de verdade acontece

Fuzzer acha superfície; o proxy acha o **comportamento**.

1. Configure o browser com proxy `127.0.0.1:8080` (use FoxyProxy).
2. Deixe o Burp **passivo**: navegue pela aplicação inteira logado. O Site Map vira seu mapa de endpoints.
3. Use **Repeater** para testar cada input manualmente e observar a diferença de resposta.
4. Use **Intruder** (ou ffuf) só depois de entender o formato da requisição.
5. Ative **Logger** para não perder nenhuma requisição feita pelo browser.

```bash
# Alternativa headless: spidar pelo ZAP
zap-cli quick-scan -s all -r http://<DOMINIO>/
```

### 9. Autenticação, sessão e JWT

A área autenticada é 50% da superfície e você só a vê depois de entrar.

- **Fluxo de login:** testar usuário inexistente vs senha errada → mensagem diferente = enumeração de usuário.
- **Registro:** cria conta própria. Duas contas sua = teste de IDOR e de lógica.
- **Reset de senha:** host header injection, token previsível, vazamento do token na resposta.
- **Cookies:** verifique `HttpOnly`, `Secure`, `SameSite` e se o valor é previsível/base64.
- **JWT:** decodifique a estrutura e teste as falhas clássicas.

```bash
# Decodifica header e payload de um JWT (sem validar assinatura)
cut -d. -f1,2 <<< "<JWT>" | tr '.' '\n' | base64 -d 2>/dev/null

# Testes de JWT, em ordem de custo:
#  1. alg:none            -> troca "alg" para none e remove a assinatura
#  2. alg HS256 com chave pública como segredo
#  3. quebra de segredo fraco com wordlist
hashcat -a 0 -m 16500 jwt.txt ~/wordlists/rockyou.txt
#  4. alterar "kid" para path traversal / SQLi
#  5. fraqueza de confusão de algoritmo (RS256 -> HS256)
```

Após logar, **repita os passos 4, 6 e 7 com o cookie de sessão**. Endpoints autenticados escondem as funções mais perigosas (upload, admin, export, debug).

### 10. Mapeamento e priorização de endpoints

Monte a tabela abaixo no final. Ela é o produto da fase e a entrada da Fase 06.

| Endpoint | Método | Parâmetros | Auth? | Observação |
|----------|--------|------------|-------|------------|
| `/api/user?id=` | GET | `id` | sim | ID sequencial → IDOR provável |
| `/upload` | POST | `file` | sim | Aceita `.php`? testar |
| `/search?q=` | GET | `q` | não | Reflete input → XSS/SQLi |
| `/download?file=` | GET | `file` | não | Nome de arquivo → LFI |

### 11. Sinal → vulnerabilidade provável

Use esta tabela como atalho mental na priorização:

| Sinal observado | Hipótese a testar | Fase |
|-----------------|-------------------|------|
| `?id=`, `?user=`, `?doc=` numérico | IDOR, SQLi | 06 / 23 |
| `?file=`, `?page=`, `?path=`, `?include=` | LFI / Path Traversal | 20 |
| Input refletido em HTML/atributo/JS | XSS | 19 |
| Error de SQL, aspas quebrando página | SQLi | 17 |
| Upload de arquivo / avatar / import | File Upload → webshell | 21 |
| `ping`, `host`, `dns`, `exec` em formulário | Command Injection | 22 |
| `XML`, `SOAP`, `Content-Type: text/xml` | XXE / SSRF | 23 |
| URL/`callback`/`webhook` como parâmetro | SSRF | 23 |
| Redirect com parâmetro `next`/`url` | Open Redirect | 23 |
| Cookie/claim de `role`, `admin`, `isAdmin` | Bypass de authz, mass assignment | 06 |
| `.git/`, `.env`, `backup.zip` | Código-fonte → credenciais | 06 |
| Versão antiga de CMS/plugin | CVE pública | 06 |

**Prioridade:** (1) exposto sem auth, (2) dá execução de código, (3) dá acesso a dados de outros usuários, (4) só ruído. Documente tudo, mas só vá fundo no que move o ataque.

## 🧰 Ferramentas

| Ferramenta | O que faz | Quando usar |
|-----------|-----------|-------------|
| `whatweb` | Fingerprint de CMS/framework/libs | Primeiro contato |
| `curl` | Requisição crua, controle total dos headers | Sempre; confirmar o que o proxy mostrou |
| `gobuster` | Fuzz de dirs, vhosts, DNS | Passe rápido e simples |
| `feroxbuster` | Fuzz recursivo com auto-filtro | Árvore de diretórios profunda |
| `ffuf` | Fuzz universal (dir, vhost, param, valor) | Filtros finos, automação |
| `arjun` | Descobre parâmetros ocultos | Antes de testar IDOR/SQLi |
| `katana` | Crawler com execução de JS | Achar rotas que wordlist não tem |
| `hakrawler` | Crawler leve em pipeline | Combinar com `gau`/`waybackurls` |
| `gau` / `waybackurls` | URLs históricas | Endpoints removidos mas vivos |
| `Burp Suite` | Proxy, Repeater, Intruder, Scanner | Toda a fase 05 manual |
| `ZAP` | Proxy open-source | Alternativa/complemento ao Burp |
| `nmap` NSE | `http-enum`, `http-headers` | Triagem rápida |

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| `curl -I` do IP dá redirect para domínio novo | Adicione em `/etc/hosts` e refaça o passo 1 |
| Resposta default idêntica para tudo | Fuzz de vhost com `-fs` do tamanho default |
| `robots.txt` com `Disallow` | Teste cada entrada manualmente |
| `.git/` acessível | `git-dumper http://<DOMINIO>/.git/ ./src` e leia o código |
| Encontrou `.php`/`.aspx`/`.jsp` | Extension fuzzing e teste de upload |
| App é SPA (React/Vue/Angular) | Leia os bundles JS; fuzze as rotas da API, não o HTML |
| Encontrou `/api/` | Enumere versões (`/api/v1`, `/v2`) e busque `swagger.json` |
| Login responde diferente por usuário | Enumeração de usuário → recolher padrão de nomes |
| Login aceita credencial fraca | Vá para Fase 06 / CPTS 16 (brute force) com cuidado de lockout |
| Cookie base64/JSON | Decodifique e teste manipulação (role, id, tenant) |
| JWT no header/cookie | Teste `alg:none`, segredo fraco, `kid` |
| Parâmetro numérico sequencial | Teste IDOR trocando +1/-1 e comparando respostas |
| Upload disponível | Teste extensão dupla, `Content-Type`, magic bytes |
| Nada achou com wordlist grande | Refaça com wordlist específica da tecnologia e com o crawl |
| Aplicação tem `/admin` sem auth | Teste direto; se bloquear, procure bypass de caminho (`//admin`, `/./admin`) |
| WAF bloqueando fuzzing | Reduza `-t`/`-rate`, adicione `-H "User-Agent: ..."` realista |

## 📤 Evidências a registrar

- Saída bruta de cada ferramenta (`gobuster.txt`, `ffuf.json`, `katana.txt`).
- Wordlists usadas — sem isso o resultado não é reproduzível.
- Lista de vhosts resolvidos e de entradas em `/etc/hosts`.
- Tabela completa de endpoints: método, parâmetros, se exige auth, papel exigido.
- Screenshot/HTTP request cru de cada endpoint sensível que confirmar comportamento anômalo.
- Tecnologia e versões exatas (para casar com CVE depois).
- Contas criadas por você e cookies/JWT válidos usados nos testes.
- Timestamps de cada varredura (para o relatório e para provar o que foi tocado).

## 🕳️ Pegadinhas

> [!warning] Os erros que mais custam tempo nesta fase
> - Fuzzar no IP em vez do vhost: todos os resultados são do site default.
> - Ignorar `-mc all`: um `403` pode conter informação (existência de path).
> - Confiar em wordlist errada: app .NET não tem diretório nenhum da `directory-list-2.3`.
> - Não fazer o passe **autenticado**: metade da aplicação some.
> - Esquecer de testar `POST`/`PUT`/`DELETE`/`OPTIONS` — só fuzzar `GET`.
> - Não olhar os arquivos `.js`: eles contêm rotas, chaves de API e lógica de autorização no front.
> - Assumir que `robots.txt` é a lista completa — é só a lista dos descuidos.
> - Fuzzing agressivo (`-t 200`) em app legada com uma conexão HTTP: timeout e dados falsos.
> - Não registrar o que já testou: você vai testar duas vezes o mesmo parâmetro e chamar de bug.
> - Esquecer rate limit/lockout — brutalizar login pode bloquear a conta do cliente.

## ✅ Checklist

- [ ] `/etc/hosts` com todos os vhosts descobertos
- [ ] Fingerprint (tecnologia + versão) registrado
- [ ] `robots.txt`, `sitemap.xml`, `.well-known` lidos
- [ ] Diretórios e arquivos fuzzados (dir + extensões)
- [ ] Vhost/subdomínio fuzzado
- [ ] Parâmetros descobertos em GET, POST e JSON
- [ ] Crawler rodado e URLs históricas coletadas
- [ ] Arquivos `.js` analisados em busca de rotas e segredos
- [ ] Navegação completa pelo Burp (inclusive logado)
- [ ] Fluxo de auth mapeado: login, registro, reset, logout
- [ ] Cookies e JWT analisados
- [ ] Enumeração repetida **autenticado**
- [ ] Endpoints priorizados por probabilidade × impacto
- [ ] Evidências salvas e indexadas por endpoint

## 🔗 Relacionado

- [[Fase 06 - Vulnerabilidade e Exploração]]
- [[Fase 04 - Enumeração de Serviços]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[CPTS 05 - Information Gathering - Web Edition]]
