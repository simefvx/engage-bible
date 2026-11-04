---
tema: Pentest / Reconhecimento
tópico: Fase 02 — OSINT e Reconhecimento Externo
tags: [htb, cpts, engage, osint, recon, dns, subdominios, vazamentos]
dificuldade: easy
---

# 🔭 Fase 02 — OSINT e Reconhecimento Externo

> [!info] Objetivo
> Montar o mapa **público** da organização antes de tocar no alvo: domínios, subdomínios, faixas de IP/ASN, tecnologia, pessoas, e-mails, credenciais vazadas e conteúdo histórico.
> Todo subdomínio esquecido é superfície não monitorada. Todo e-mail descoberto é candidato a password spraying mais tarde.

---

## 🎯 Quando entrar

- Assim que o escopo estiver assinado e houver alvo com nome de domínio ou faixa de IP.
- Antes do scan de rede — OSINT escolhe **onde** vale escanear.
- Depois de qualquer descoberta que sugira nova marca, aquisição ou filial (a organização tem mais domínios do que declara).

> [!tip] Passivo primeiro, ativo depois
> Coleta passiva não toca no alvo e não gera log no cliente. Só depois de fechar a lista de alvos é que se parte para o ativo (brute de DNS, port scan, vhost fuzz) — este sim gera tráfego ao alvo e precisa estar dentro da janela.

---

## 📥 Pré-requisitos e inputs

| Input | Origem | Uso |
|-------|--------|-----|
| Domínio(s) raiz do escopo | Pré-engajamento | Ponto de partida de DNS e certificados |
| Nome legal da empresa | Contrato | WHOIS reverso, registros, LinkedIn |
| Faixas de IP / ASN | Cliente ou descoberta | Mapear infraestrutura própria vs terceiros |
| Chaves de API | Você (gratuito/pago) | Shodan, Censys, HIBP, Hunter |
| Regras do RoE | Fase 01 | Limita brute force e rate |

Ferramentas externas exigem chave/conta: crie e guarde em `00-admin/keys.md` (o arquivo fora do relatório).

---

## 🪜 Passo a passo

### 1. WHOIS, registros e dono do domínio

```bash
# WHOIS do domínio: registrante, nameservers, datas, contatos administrativos
whois <DOMINIO> | grep -Ei 'registrar|name server|creat|expir|org|admin'

# WHOIS reverso por e-mail/organização (descobre domínios irmãos)
# via serviços web (ViewDNS, WhoisXMLAPI) ou:
amass intel -whois -d <DOMINIO>
```

O que importa: nameservers (provedor de DNS revela stack), datas (empresa nova = infra nova), e-mail de registro (pode virar alvo de OSINT de pessoa).

### 2. Resolução DNS e infraestrutura de e-mail

```bash
# Registros principais — IPs, nameservers e infra de correio
dig +short A <DOMINIO>            # IPv4
dig +short AAAA <DOMINIO>         # IPv6 (frequentemente fora do escopo!)
dig +short NS <DOMINIO>           # provedor de DNS
dig +short MX <DOMINIO>           # provedor de e-mail (M365? Google? on-prem?)

# SPF e DMARC revelam serviços autorizados (SendGrid, Mailchimp, IPs de saída)
dig +short TXT <DOMINIO>
dig +short TXT _dmarc.<DOMINIO>
```

> [!tip] SPF é um mapa grátis
> O registro SPF lista os IPs/serviços que enviam e-mail em nome da empresa. Isso revela SaaS e faixas de IP legítimas do cliente.

### 3. Transferência de zona (AXFR)

```bash
# Pede o dump completo da zona ao nameserver — se permitido, entrega tudo de uma vez
dig axfr @<NS> <DOMINIO>

# Fallback com host/dnsrecon quando o dig não basta
host -t axfr <DOMINIO> <NS>
dnsrecon -d <DOMINIO> -t axfr
```

AXFR aberto é misconfiguração clássica em DNS interno/legado. Se funcionar, você tem todos os hosts de uma vez.

### 4. Certificate Transparency (subdomínios sem tocar no alvo)

```bash
# crt.sh devolve todo nome que já apareceu em certificado TLS — passivo e completo
curl -s 'https://crt.sh/?q=%25.<DOMINIO>&output=json' \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' | tr '[:upper:]' '[:lower:]' | sort -u > subs_crt.txt

# Consultas por entidade (Organization) descobrem domínios irmãos
curl -s 'https://crt.sh/?O=<ORGANIZACAO>&output=json' | jq -r '.[].name_value' | sort -u
```

### 5. Enumeração de subdomínios (passiva + brute)

```bash
# --- Passivo (não toca no alvo) ---
subfinder -d <DOMINIO> -all -recursive -o subs_subfinder.txt
amass enum -passive -d <DOMINIO> -o subs_amass.txt
assetfinder --subs-only <DOMINIO> | sort -u > subs_assetfinder.txt

# --- Brute de DNS (ativo, gera tráfego) ---
# shuffledns usa massdns como resolver: rápido e paralelo
shuffledns -d <DOMINIO> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -r resolvers.txt -o subs_brute.txt

# gobuster dns: brute simples, tolera wildcard com --wildcard
gobuster dns -d <DOMINIO> -w \
  -t 50 --wildcard -o subs_gobuster.txt

# --- Consolida tudo em uma lista limpa ---
cat subs_*.txt | sort -u > subs_all.txt
# dnsx confirma quais realmente resolvem (remove mortos)
cat subs_all.txt | dnsx -silent -a -resp -o subs_live.txt
```

> [!warning] Permutação dá o subdomínio esquecido
> `dev-`, `stg-`, `-old`, `-bak` costumam escapar das wordlists. Gere permutações (`altdns`, `gotator`) e valide com `dnsx`.

### 6. Virtual hosts (um IP, vários sites)

```bash
# FUZZ no header Host descobre sites servidos pelo mesmo IP sem DNS público
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u https://<IP> -H "Host: FUZZ.<DOMINIO>" -fs 0,<TAMANHO-BASE> -mc all
# -fs filtra respostas do site default (o tamanho dele); todo vhost real tem tamanho diferente
```

### 7. ASN e faixas de IP

```bash
# Qual ASN anuncia o IP do alvo — e quais outras faixas ele anuncia
whois -h whois.radb.net -- '-i origin <ASN>'
amass intel -asn <ASN> -o asn_ranges.txt

# Consultas web equivalentes: bgp.he.net, bgpview.io, ipinfo.io/<IP>
```

Mapeie: faixa própria do cliente (atacável) × faixa de cloud/CDN (confirmar autorização antes de tocar).

### 8. Fingerprint de tecnologia

```bash
# whatweb: identifica CMS, servidor, JS, analytics (agressividade -a 3)
whatweb -a 3 https://<DOMINIO>

# Headers crus: server, X-Powered-By, cookies de framework, redirects
curl -sSI https://<DOMINIO>
curl -sS -D- -o /dev/null https://<DOMINIO>

# Hash do favicon para bater no Shodan (o mesmo app gera o mesmo hash na org toda)
python3 -c "import mmh3,requests,codecs; r=requests.get('https://<DOMINIO>/favicon.ico'); \
print(mmh3.hash(codecs.encode(__import__('base64').b64encode(r.content),'rot13')))"
```

| Pista | O que revela |
|-------|--------------|
| `Server:` | nginx/Apache/IIS + versão quando mal configurado |
| `X-Powered-By` | PHP/ASP.NET/Express |
| `Set-Cookie` | Framework (JSESSIONID, ASP.NET_SessionId, PHPSESSID) |
| Favicon hash | Mesma aplicação em outros hosts/portas |
| `robots.txt` / `sitemap.xml` | Áreas "escondidas" declaradas pelo dev |
| Erros (`/nonexistent`) | Stack completa, paths, versões |

### 9. Pessoas, e-mails e padrão de nomes

```bash
# theHarvester: e-mails, hosts e nomes de fontes públicas
theHarvester -d <DOMINIO> -b all -l 500 -f harvester.html
```

Do que se extrai valor real: **padrão de e-mail** (`nome.sobrenome@`, `nome@`, `n.sobrenome@`) e **lista de usuários**. Vêm de LinkedIn, assinaturas, PDFs públicos, commits de Git. Gere a lista combinando nomes (FLast, First.Last) para usar em SMTP enum, Kerberos e spraying.

| Fonte | O que dá |
|-------|----------|
| LinkedIn | Nomes, cargos, organograma |
| Site institucional / press releases | Nomes de diretores |
| Google dorks `@<DOMINIO> filetype:pdf` | Documentos com e-mails |
| Padrão de e-mail confirmado | Lista de usuários para sprays |

### 10. Vazamentos de credencial

```bash
# HIBP: contas comprometidas (precisa de API key)
curl -s -H "hibp-api-key: <KEY>" -H "user-agent: engage" \
  'https://haveibeenpwned.com/api/v3/breachedaccount/<EMAIL>?truncateResponse=false'

# HIBP: senhas do domínio no dump (pwned domain)
curl -s -H "hibp-api-key: <KEY>" 'https://haveibeenpwned.com/api/v3/breacheddomain/<DOMINIO>'
```

Outras fontes: DeHashed (pago), Intelligence X, pastebins, dumps públicos. Cruze e-mail vazado + senha antiga = hipótese forte de reuso. **Nunca** teste a senha vazada sem autorização explícita de password attack no RoE.

### 11. GitHub, dorks e segredos expostos

```bash
# GitHub Code Search via API (precisa de token): busca segredo no código da org
curl -s -H "Authorization: token <GITHUB-TOKEN>" -H "Accept: application/vnd.github+json" \
  'https://api.github.com/search/code?q=org%3A<ORG>+password'

# Varredura de repositório clonado por segredos
trufflehog git file://./repo --only-verified
gitleaks detect --source ./repo --report-format json -r leaks.json
```

Google dorks úteis:

| Dork | Alvo |
|------|------|
| `site:<DOMINIO> filetype:pdf` | Documentos, e-mails, metadados |
| `site:<DOMINIO> intitle:"index of"` | Diretórios listados |
| `site:<DOMINIO> ext:php intitle:phpinfo` | `phpinfo()` exposto |
| `site:github.com "<DOMINIO>"` | Código vazado da org |
| `"<DOMINIO>" inurl:admin OR inurl:login` | Painéis administrativos |
| `filetype:xls site:<DOMINIO>` | Planilhas (nomes, estrutura) |

### 12. Shodan, Censys e exposição de serviços

```bash
# Shodan CLI: o que o mundo já vê do ASN/org do cliente, sem você escanear nada
shodan init <SHODAN-KEY>
shodan search 'org:"<ORGANIZACAO>" port:3389' --fields ip_str,port,product,hostnames
shodan host <IP>                       # banners já conhecidos do IP
shodan search 'ssl.cert.subject.CN:"<DOMINIO>"'   # hosts por certificado

# Censys CLI (equivalentes) — bom para certificados e vhosts
censys search 'services.tls.certificates.leaf_data.names: <DOMINIO>'
```

Priorize achados: RDP/SMB/DB expostos, painéis de administração, versões desatualizadas, certificado com muitos SANs.

### 13. Conteúdo histórico e URLs antigas

```bash
# Wayback Machine CDX: todas as URLs arquivadas (inclui páginas e endpoints removidos)
curl -s 'http://web.archive.org/cdx/search/cdx?url=*.<DOMINIO>/*&output=text&fl=original&collapse=urlkey' \
  | sort -u > wayback_urls.txt

# Ferramentas que agregam Wayback + CommonCrawl + OTX
waybackurls <DOMINIO> > wayback2.txt
gau --subs <DOMINIO> > gau.txt

# Filtra o que tem parâmetro (candidatos a IDOR/SQLi/LFI) e o que é interessante
cat gau.txt | grep '=' | sort -u > urls_param.txt
cat gau.txt | grep -Ei '\.(bak|old|zip|sql|conf|env|git)' | sort -u
```

### 14. Metadados de documentos

```bash
# Baixa e extrai autor, software, caminhos internos, nomes de usuário dos PDFs/DOCs
exiftool -a -u -g1 documento.pdf

# metagoofil: baixa documentos públicos do domínio e extrai metadados em lote
metagoofil -d <DOMINIO> -t pdf,docx,xlsx -l 50 -n 20 -o metadocs/
```

Caminhos tipo `C:\Users\<USUARIO>\` nos metadados entregam login real do Windows — ouro para sprays.

### 15. Buckets e armazenamento em cloud

```bash
# Testa se um bucket S3 é listável sem credencial
aws s3 ls s3://<BUCKET> --no-sign-request
aws s3 sync s3://<BUCKET> ./bucket-loot --no-sign-request

# Enumeração de buckets por variação de nome (empresa-dev, empresa-backup, empresa-assets)
cloud_enum -k <DOMINIO> -k <EMPRESA> --quickscan
```

Nomes de bucket são previsíveis: `<empresa>-backup`, `<empresa>-dev`, `<empresa>-uploads`. Verifique cada variação.

### 16. Consolidação

Monte um único artefato `recon/resumo.md`:

| Campo | Exemplo |
|-------|---------|
| Domínios raiz | example.com, example.net |
| Subdomínios vivos | 214 (lista em `subs_live.txt`) |
| Faixas de IP | 203.0.113.0/24 (própria), CDN (fora) |
| Tecnologia | nginx 1.24, WordPress 6.x, M365 |
| E-mails/usuários | 47 (padrão `nome.sobrenome@`) |
| Vazamentos | 6 contas em 3 breaches |
| Exposição | RDP 3389 em 2 IPs (Shodan) |
| Hipóteses | spray contra M365; WP antigo; bucket aberto |

---

## 🧰 Ferramentas

| Ferramenta | Função | Instalação |
|------------|--------|-----------|
| `whois`, `dig`, `host` | DNS e registro de domínio | nativo |
| `dnsrecon`, `dnsenum`, `fierce` | DNS, AXFR, brute | pacman |
| `subfinder`, `amass`, `assetfinder` | Subdomínios (passivo/ativo) | `go install` / pacman |
| `shuffledns`, `massdns`, `puredns` | Brute DNS de alta performance | pacman / `go install` |
| `dnsx` | Resolver/validar subdomínios e respostas | pacman |
| `gobuster`, `ffuf` | Brute DNS e vhost fuzz | pacman |
| `whatweb`, `wappalyzer` | Fingerprint de tecnologia | pacman / extensão |
| `theHarvester` | E-mails, hosts, nomes | pacman |
| `shodan`, `censys` | Exposição e banners (passivo) | pip |
| `gau`, `waybackurls`, `katana` | URLs históricas e crawling | pacman / `go install` |
| `exiftool`, `metagoofil` | Metadados de documentos | pacman |
| `cloud_enum`, `s3scanner` | Buckets/cloud storage aberto | pip / pacman |
| `trufflehog`, `gitleaks` | Segredos em repositórios | pacman |

---

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| AXFR responde | Consolidar a zona inteira; tratar como achado (misconfig de DNS) |
| Subdomínio aponta para cloud/CDN de terceiro | Confirmar autorização; se fora, marcar e excluir do scan |
| Subdomínio com certificado expirado ou vhost default | Candidato forte a app esquecida — priorizar enumeração |
| E-mail com padrão confirmado | Montar lista de usuários para SMTP enum/spraying |
| Credencial vazada encontrada | Registrar; só testar se o RoE autorizar password attack |
| Bucket S3 listável | Documentar (achado de severidade alta); baixar apenas amostra, sem exfiltrar PII |
| `robots.txt`/Wayback com `/admin`, `.old`, `.bak` | Priorizar na fase web |
| Serviço crítico exposto (RDP/SMB/DB) via Shodan | Alto valor: validar na fase 03/04 |
| Nada além do domínio raiz | Ir para brute de DNS + CT logs + permutações antes de desistir |

---

## 📤 Evidências a registrar

- Lista consolidada de domínios e subdomínios vivos (arquivo + data da coleta).
- Consultas DNS relevantes (`dig axfr`, MX, SPF) com saída original.
- Saída de crt.sh, subfinder/amass e brute (para provar cobertura).
- Fingerprint: `whatweb`, headers, hash de favicon.
- Lista de e-mails/usuários e o **padrão** identificado.
- Achados de vazamento: fonte, data, dado comprometido (mascarado).
- Segredos/tokens encontrados (referência, nunca valor completo).
- Exposição via Shodan/Censys com IP, porta e data da consulta.
- URLs históricas com parâmetro e arquivos sensíveis antigos.
- Metadados que revelaram usuário ou path interno.

---

## 🕳️ Pegadinhas

- 🕳️ **Confundir CDN com infra do cliente.** O IP do Cloudflare/Akamai não é alvo.
- 🕳️ **IPv6 esquecido.** `AAAA` pode apontar para um host fora do escopo ou não monitorado.
- 🕳️ **Brute DNS sem rate limit.** Resolver público grande + wildcard = resultados falsos e possível blocklist.
- 🕳️ **Wildcard DNS.** Todo `qualquercoisa.dominio` resolve; filtrar por tamanho/IP do vhost default.
- 🕳️ **Confiar só em uma fonte.** crt.sh não vê subdomínio sem certificado; brute não vê o que não está na wordlist.
- 🕳️ **Testar senha vazada sem autorização.** Vira incidente (conta bloqueada) e viola RoE.
- 🕳️ **Baixar bucket inteiro com PII.** Exfiltração desnecessária; quebra contrato e lei.
- 🕳️ **Trustar metadata de documento sem confirmar o autor real.** Nomes podem estar duplicados/antigos.
- 🕳️ **Ignorar o passivo e já começar escaneando.** Gera alerta no cliente cedo e perde a vantagem da surpresa.
- 🕳️ **Não datar a coleta.** Tecnologia e exposição mudam; sem data, a evidência é inútil.

---

## ✅ Checklist

- [ ] WHOIS e nameservers documentados
- [ ] Registros A/AAAA/NS/MX/TXT/SPF/DMARC coletados
- [ ] AXFR testado em todos os nameservers
- [ ] crt.sh + subfinder + amass consolidados
- [ ] Brute DNS executado com wordlist relevante
- [ ] Permutações testadas (`dev-`, `stg-`, `-old`)
- [ ] Subdomínios validados com `dnsx` (vivos)
- [ ] Vhost fuzz executado nos IPs do escopo
- [ ] ASN e faixas de IP mapeados
- [ ] Fingerprint (`whatweb`, headers, favicon) registrado
- [ ] E-mails/usuários e padrão de nomes definidos
- [ ] HIBP/vazamentos consultados
- [ ] Dorks (Google/GitHub) executados
- [ ] Shodan/Censys consultados
- [ ] Wayback/gau com URLs paramétricas salvas
- [ ] Metadados de documentos extraídos
- [ ] Buckets de cloud testados
- [ ] Resumo do recon escrito com hipóteses priorizadas

---

## 🔗 Relacionado

- [[Workflow do Engage]]
- [[CPTS 05 - Information Gathering - Web Edition]]
- [[CPTS 04 - Footprinting]]
- [[Fase 01 - Pré-Engajamento]]
- [[Fase 03 - Reconhecimento de Rede]]
