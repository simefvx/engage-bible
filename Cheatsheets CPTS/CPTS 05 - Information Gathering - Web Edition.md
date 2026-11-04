---
tema: Pentest / Reconhecimento
tópico: Information Gathering — Web Edition
tags: [htb, cpts, osint, recon, dns, subdominios, vhost, fingerprint, crawler, dorks, shodan, censys, buckets]
dificuldade: easy
---

# CPTS 05 - Information Gathering - Web Edition — Cheatsheet

> [!info] Escopo deste cartão
> Reconhecimento **externo** de uma aplicação web: DNS e subdomínios, virtual hosts, fingerprint de tecnologia, headers, crawling, URLs históricas, Google/GitHub dorks, motores de OSINT (Shodan/Censys) e buckets de cloud expostos.
> Regra de ouro: **sem tocar no alvo de forma agressiva** — quase tudo aqui é passivo ou levemente ativo.

---

## 🎯 Quando usar

| Situação | Seção |
|----------|-------|
| Tenho só um domínio e nada mais | WHOIS + DNS + Certificate Transparency |
| Preciso da superfície completa de subdomínios | Enumeração de subdomínios |
| Um servidor responde igual para qualquer host | Virtual host fuzzing |
| Não sei o que roda ali | Fingerprint de tecnologia |
| Preciso de endpoints antigos/escondidos | Crawling + Wayback/gau |
| Quero credenciais vazadas | Google dorks + GitHub |
| Quero infraestrutura exposta sem scan | Shodan + Censys |
| Suspeito de storage mal configurado | Buckets de cloud |

---

## 🧰 Ferramentas

| Ferramenta | Função |
|------------|--------|
| `whois` | Registrante, ASN, netblocks, nameservers |
| `dig`, `host`, `nslookup` | Consultas DNS |
| `dnsenum`, `dnsrecon`, `fierce` | Enumeração DNS completa |
| `subfinder`, `amass` | Descoberta de subdomínios |
| `gobuster dns`, `ffuf`, `dnsx` | Brute force de subdomínio |
| `ffuf`, `gobuster vhost` | Virtual host fuzzing |
| `whatweb`, `webanalyze`, `wafw00f` | Fingerprint e WAF |
| `curl` | Headers, robots, sitemap, favicon |
| `katana`, `hakrawler`, `gospider` | Crawling da aplicação |
| `gau`, `waybackurls` | URLs históricas |
| `httpx` | Validar quais hosts/URLs realmente respondem |
| `trufflehog`, `gitleaks`, `gitdorker` | Segredos em repositórios |
| `cloud_enum`, `s3scanner`, `aws` CLI | Buckets S3/Azure/GCS |
| Shodan, Censys, crt.sh | Interfaces web de OSINT |

---

## 📇 WHOIS

```bash
# Registrante, datas, nameservers e contatos técnicos do domínio.
# O que extrair: organização, e-mails, país, provedor de DNS.
whois <DOMINIO>

# Blocos de IP, ASN e contatos de abuse da faixa.
whois <IP>

# Consultas a servidores específicos quando o registro é delegado.
whois -h whois.arin.net <IP>     # ARIN (América)
whois -h whois.ripe.net <IP>     # RIPE (Europa)
whois -h whois.apnic.net <IP>    # APNIC (Ásia-Pacífico)
```

**O que extrair:** netblocks da organização (varrer depois), servidores de DNS autoritativos, e-mails de contato (padrão de nome de usuário), datas de criação/expiração.

---

## 🌐 DNS

| Comando | O que faz |
|---------|-----------|
| `dig A <DOMINIO>` | Endereço IPv4 — primeiro passo |
| `dig AAAA <DOMINIO>` | IPv6 (muitas vezes desprotegido) |
| `dig MX <DOMINIO>` | Servidores de e-mail (mail provider, terceiros) |
| `dig NS <DOMINIO>` | Nameservers autoritativos |
| `dig TXT <DOMINIO>` | SPF, DKIM, verificação de serviços, chaves |
| `dig CNAME <DOMINIO>` | Aliases — revelam SaaS e CDN |
| `dig SRV _ldap._tcp.<DOMINIO>` | Serviços internos (DCs, SIP, autodiscover) |
| `dig -x <IP>` | PTR — nome reverso de um IP |
| `dig +trace <DOMINIO>` | Resolve passo a passo desde a raiz |
| `dig AXFR <DOMINIO> @<NS>` | Transferência de zona (dump completo) |
| `dig +short <DOMINIO>` | Saída limpa para scripts |

```bash
# Coleta rápida de tudo que costuma revelar infra
dig A <DOMINIO>      ;  dig MX <DOMINIO>  ;  dig NS <DOMINIO>
dig TXT <DOMINIO>    ;  dig -x <IP>

# Tenta transferência de zona em cada nameserver (falha clássica de config)
for ns in $(dig +short NS <DOMINIO>); do dig AXFR <DOMINIO> @$ns; done
```

---

## 🧭 Enumeração DNS automatizada

| Comando | O que faz |
|---------|-----------|
| `dnsenum --enum <DOMINIO> -f lista.txt -r` | Zona, brute recursivo, reverse lookup, WHOIS |
| `dnsrecon -d <DOMINIO> -t std` | Enumeração padrão (NS, SOA, MX, SRV, TXT) |
| `dnsrecon -d <DOMINIO> -t brt -D lista.txt` | Brute force de subdomínios |
| `dnsrecon -d <DOMINIO> -t axfr` | Testa transferência de zona |
| `dnsrecon -d <DOMINIO> -t zonewalk` | NSEC zone walking (DNSSEC) |
| `fierce --domain <DOMINIO>` | Descoberta rápida de subdomínios e rede |

```bash
# Cycle completo: registros + brute + reverse + WHOIS
dnsenum --enum <DOMINIO> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r

# Só transferência de zona, para confirmar rápido
dnsrecon -d <DOMINIO> -t axfr
```

---

## 🔎 Subdomínios

| Comando | O que faz |
|---------|-----------|
| `subfinder -d <DOMINIO> -all -o subs.txt` | Coleta passiva em muitas fontes |
| `amass enum -passive -d <DOMINIO> -o amass.txt` | Enumeração passiva profunda |
| `amass enum -active -d <DOMINIO>` | Inclui resolução e brute |
| `gobuster dns -d <DOMINIO> -w lista.txt -r <NS>` | Brute de subdomínio |
| `ffuf -w lista.txt -u http://<NS> -H "Host: FUZZ.<DOMINIO>" -fs <tam>` | Descobre subdomínios que só existem por Host header |
| `httpx -l subs.txt -sc -title -tech-detect -o vivos.txt` | Valida o que existe e mostra status/título/tech |

```bash
# Certificate Transparency: todo certificado emitido aparece aqui
curl -s "https://crt.sh/?q=%25.<DOMINIO>&output=json" \
  | jq -r '.[].name_value' | sed 's/\*\.//g' | sort -u > crt_subs.txt

# Pipeline completo: descobrir -> validar -> tirar screenshot
subfinder -d <DOMINIO> -silent | httpx -sc -title -tech-detect

# Consolidar várias fontes e remover duplicados
cat subs.txt crt_subs.txt | sort -u > all_subs.txt
```

> 💡 **crt.sh exige só o domínio**, sem tocar no alvo. É a fonte passiva mais rentável depois do `subfinder`.

---

## 🏠 Virtual host fuzzing

Servidor único com vários sites: o conteúdo muda conforme o header `Host`.

```bash
# Fuzz do Host header. -fs filtra pelo tamanho da resposta padrão
# (descubra o tamanho da página default primeiro, senão o resultado é ruído).
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
     -u http://<IP> -H "Host: FUZZ.<DOMINIO>" -fs 1234

# Alternativa com validação de domínio completo.
gobuster vhost -u http://<IP> -w lista.txt --append-domain --exclude-length 1234
```

**O que extrair:** painéis de admin internos, apps de staging, sites antigos, APIs não publicadas no DNS público.

---

## 🔬 Fingerprint de tecnologia

| Comando | O que faz |
|---------|-----------|
| `whatweb -v <URL>` | CMS, servidor, linguagem, headers, versões |
| `whatweb -a 3 <URL>` | Agressividade máxima (mais plugins ativos) |
| `webanalyze -host <URL> -crawl 2` | Fingerprint em lote/automatizado |
| `wafw00f <URL>` | Detecta WAF e o fabricante |
| `curl -I <URL>` | Só os headers de resposta |
| `curl -s -D - <URL> -o /dev/null` | Headers + descarta o corpo |
| `nmap -p80,443 --script http-headers,http-title,http-favicon <IP>` | Título, headers e favicon |

```bash
# Headers que entregam a stack
curl -sI https://<DOMINIO>
#   Server, X-Powered-By, X-AspNet-Version, X-Generator, Set-Cookie (favorece linguagem)

# Favicon hash: identifica o produto em escala via Shodan
curl -s http://<IP>/favicon.ico | md5sum
# Depois: shodan search 'http.favicon.hash:<HASH_MMH3>'
```

> 💡 Cookie de sessão denuncia stack: `PHPSESSID` (PHP), `JSESSIONID` (Java), `ASP.NET_SessionId`, `csrftoken`/`sessionid` (Django), `_rails` (Ruby).

---

## 🕷️ Crawling e URLs históricas

| Comando | O que faz |
|---------|-----------|
| `katana -u <URL> -d 3 -jc -o katana.txt` | Crawler moderno; `-jc` segue JavaScript |
| `hakrawler -url <URL> -depth 2 -plain` | Crawler rápido via stdin/stdout |
| `gospider -s <URL> -d 3 -o out/` | Crawler concorrente com extração de links |
| `gau <DOMINIO> \| tee urls.txt` | URLs conhecidas (Wayback, Common Crawl, OTX) |
| `waybackurls <DOMINIO>` | URLs do Wayback Machine |
| `curl "http://web.archive.org/cdx/search/cdx?url=<DOMINIO>/*&output=text&fl=original&collapse=urlkey"` | Consulta direta ao CDX do Wayback |

```bash
# Coletar e filtrar o que interessa
gau <DOMINIO> | tee gau.txt
grep -Ei '\.(js|json|php|aspx|txt|bak|old|zip|env)$' gau.txt | sort -u

# Só as URLs que ainda respondem 200
cat gau.txt | httpx -mc 200 -o vivas.txt

# Procurar parâmetros (potenciais pontos de injeção)
grep '=' gau.txt | sort -u
```

**O que extrair:** endpoints esquecidos, arquivos `.bak`/`.old`, `.git`, parâmetros com nome sugestivo (`debug`, `admin`, `file`, `url`), APIs internas, formulários antigos.

---

## 🔍 Google dorks

| Operador | Para que serve |
|----------|----------------|
| `site:` | Restringe ao domínio (ou exclui com `-site:`) |
| `inurl:` | Termo presente na URL |
| `intitle:` | Termo no título da página |
| `intext:` | Termo no corpo |
| `filetype:` / `ext:` | Extensão do arquivo |
| `cache:` | Versão em cache (histórico) |
| `related:` | Sites semelhantes |
| `-` | Exclui termo |
| `OR` / `\|` | Alternativa |

```text
site:<DOMINIO> -www
site:<DOMINIO> inurl:admin | inurl:login | inurl:dashboard
site:<DOMINIO> ext:php | ext:aspx | ext:jsp
site:<DOMINIO> intitle:"index of" "backup" | "config"
site:<DOMINIO> filetype:pdf | filetype:xls | filetype:sql
site:<DOMINIO> inurl:"/.git" | inurl:".env" | inurl:"id_rsa"
site:pastebin.com "<DOMINIO>"
site:github.com "<DOMINIO>" password
site:trello.com "<DOMINIO>"
```

**O que extrair:** diretórios listáveis ("index of"), planilhas com dados internos, painéis de login, repositórios, dumps em pastebin.

---

## 🐙 GitHub e vazamentos de código

```text
# Dorks clássicos na busca do GitHub
"<DOMINIO>" password
"<DOMINIO>" api_key | apikey | secret
"<ORGANIZACAO>" filename:.env
"<ORGANIZACAO>" filename:id_rsa
"<DOMINIO>" "BEGIN RSA PRIVATE KEY"
"<DOMINIO>" "AKIA"          # access key AWS
```

```bash
# Varredura de repositório clonado procurando segredos
gitleaks detect --source . -v
trufflehog filesystem . --only-verified

# Histórico completo do Git (segredo removido ainda vive no commit antigo)
git log -p | grep -Ei 'password|secret|api[_-]?key|token'
```

**O que extrair:** chaves de API, credenciais de banco, tokens, arquivos `.env`, chaves privadas, endpoints internos.

---

## 🛰️ Shodan e Censys

```text
# Shodan (web/CLI)
hostname:<DOMINIO>
org:"<ORGANIZACAO>"
net:<CIDR>
ssl.cert.subject.cn:<DOMINIO>
http.favicon.hash:<HASH_MMH3>
product:"Apache httpd" country:"BR"

# Censys
services.tls.certificates.leaf_data.subject.common_name: <DOMINIO>
services.http.response.html_title: "<TITULO>"
```

```bash
# CLI do Shodan
shodan search 'hostname:<DOMINIO>'
shodan host <IP>                 # portas, banners e vulns conhecidas do IP
shodan stats --facets port 'org:"<ORGANIZACAO>"'
```

**O que extrair:** portas expostas na internet, banners, certificados (nomes internos!), hosts esquecidos não cobertos pelo inventário, serviços de terceiros na mesma faixa.

---

## 📄 Arquivos e rotas padrão

```bash
# robots.txt: revela o que o site NÃO quer indexar (ou seja, o que interessa)
curl -s http://<IP>/robots.txt

# Sitemap: inventário de URLs que o dono publicou
curl -s http://<IP>/sitemap.xml
curl -s http://<IP>/sitemap_index.xml | grep -o '<loc>.*</loc>'

# Contatos de segurança / divulgação
curl -s http://<IP>/.well-known/security.txt

# Fichas e metadados
curl -s http://<IP>/humans.txt
curl -s http://<IP>/.git/HEAD        # se 200, o repositório está exposto

# Cabeçalhos e redirecionamentos
curl -sI http://<IP>/
curl -s -L -o /dev/null -w '%{url_effective}\n' http://<IP>/
```

**O que extrair:** `Disallow:` (áreas sensíveis), `Allow:`, caminhos de admin, APIs internas, `.git` exposto (dump completo do código).

---

## 🪣 Buckets de cloud

```bash
# S3: bucket público responde listagem em XML
curl -s http://<BUCKET>.s3.amazonaws.com/
curl -s https://<BUCKET>.s3.<REGIAO>.amazonaws.com/?list-type=2

# Sem credencial, com AWS CLI
aws s3 ls s3://<BUCKET> --no-sign-request
aws s3 cp s3://<BUCKET>/arquivo.txt . --no-sign-request
aws s3 sync s3://<BUCKET> ./loot --no-sign-request

# Azure Blob: listagem de container
curl -s "https://<CONTA>.blob.core.windows.net/<CONTAINER>?restype=container&comp=list"

# Google Cloud Storage
curl -s https://storage.googleapis.com/<BUCKET>/

# Força bruta de nomes por palavra-chave da empresa
cloud_enum -k <EMPRESA> -k <DOMINIO>
```

**O que extrair:** backups, dumps de banco, arquivos de config, código-fonte, dados de clientes (PII — cuidado com tratamento e reporte).

---

## ⚠️ Pegadinhas

| 🕳️ | Detalhe |
|-----|---------|
| Confundir wildcard DNS | `*.dominio` responde tudo; filtre por tamanho/status para não achar falso subdomínio. |
| `dig AXFR` contra o alvo errado | Precisa consultar o **nameserver** autoritativo, não o site. |
| Scan agressivo em OSINT | Aqui é reconhecimento: prefira fontes passivas antes de tocar no alvo. |
| Confiar só no `whatweb` | Header pode ser mascarado; cruze com favicon, cookies e comportamento. |
| Vhost fuzz sem baseline | Sem medir o tamanho da resposta default, todo resultado parece válido. |
| `gau` como verdade | URLs históricas podem estar mortas; valide com `httpx`. |
| Baixar bucket inteiro | Só pegue o necessário para provar impacto; o resto é dado de terceiro. |
| Esquecer IPv6 | `AAAA` pode apontar para host sem o mesmo hardening do IPv4. |
| Certificate Transparency ignorado | É passivo, gratuito e frequentemente revela o subdomínio interno. |
| Segredo achado no GitHub sem contexto | Confirme se ainda é válido **antes** de reportar; muitos já foram rotacionados. |

---

## ✅ Checklist rápido

- [ ] WHOIS do domínio e dos netblocks obtido
- [ ] Registros A, AAAA, MX, NS, TXT, CNAME, SRV coletados
- [ ] Transferência de zona testada em cada NS
- [ ] Subdomínios consolidados de várias fontes (`subfinder`, `amass`, `crt.sh`)
- [ ] Hosts vivos validados com `httpx` (status/título/tech)
- [ ] Virtual host fuzzing feito com baseline de tamanho
- [ ] Fingerprint de tecnologia + WAF identificado
- [ ] Headers, cookies, `robots.txt`, `sitemap.xml`, `security.txt` revisados
- [ ] Crawling (`katana`/`hakrawler`) e URLs históricas (`gau`/Wayback) coletadas
- [ ] Google dorks e GitHub dorks executados
- [ ] Shodan/Censys consultados por domínio, org e favicon
- [ ] Buckets S3/Azure/GCS testados
- [ ] Tudo salvo em arquivos com nome versionado e evidência

---

## 🔗 Relacionado

- [[CPTS 04 - Footprinting]]
- [[CPTS 03 - Network Enumeration with Nmap]]
- [[Fase 02 - OSINT e Reconhecimento Externo]]
- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[Fase 05 - Enumeração Web]]
