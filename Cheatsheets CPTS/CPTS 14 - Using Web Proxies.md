---
tema: Proxy de Interceptação
tópico: Uso de Burp Suite e ZAP para interceptar, modificar e automatizar tráfego HTTP
tags: [htb, cpts, burp, zap, proxy, intercept, intruder, foxyproxy, tls]
dificuldade: easy
---

# CPTS 14 — Using Web Proxies

> [!info] Objetivo
> Colocar um proxy entre o browser, o Burp ou o ZAP e a aplicação alvo para **ver, modificar e repetir** requisições. Saber configurar certificado TLS, escopo, match/replace, Intruder e fuzzing — por GUI e por linha de comando.

## 🎯 Quando usar

- Toda aplicação web do engajamento: interceptar é pré-requisito para testar parâmetros, headers e cookies.
- Quando a ferramenta de linha de comando (ffuf, sqlmap, curl) precisa passar pelo proxy para registrar o tráfego.
- Quando precisa fuzzar posições específicas de uma requisição complexa (Intruder) ou aplicar regras globais (match/replace).

## 🧰 Ferramentas

| Ferramenta | Papel |
|---|---|
| Burp Suite (Community/Pro) | Proxy, Repeater, Intruder, Decoder, Comparer, Extensions |
| OWASP ZAP | Alternativa open-source: Fuzzer, Spider, Active Scan, Break |
| FoxyProxy | Extensão de browser para alternar proxy |
| `curl`, `ffuf`, `sqlmap` | Clientes que falam HTTP via proxy |
| Proxychains | Encadear ferramentas não-HTTP no proxy (ver CPTS 12) |
| `interactsh`, Collaborator | Detecção de interações OOB |

## ⚙️ Configuração básica

Listener padrão em `127.0.0.1:8080` (Burp e ZAP usam a mesma porta por padrão — não rode os dois juntos sem trocar uma delas).

```text
Burp:  Proxy → Options → Proxy Listeners → 127.0.0.1:8080
ZAP:   Options → Network → Local Servers/Proxies → 127.0.0.1:8080
```

### FoxyProxy

```text
1. Add Proxy → Title: Burp | Type: HTTP | Host: 127.0.0.1 | Port: 8080
2. Patterns: *://*<DOMINIO>*  (só o alvo passa pelo proxy — evita poluir o histórico)
3. Habilitar/desabilitar pelo ícone da extensão.
```

**Quando usar:** se você deixar o proxy global, todo o tráfego do browser (Google, atualizações) enche o histórico e torna a busca impossível.

## 🔐 Certificado TLS

```bash
# 1. Com o proxy ligado, acesse http://burp no browser e baixe o CA Certificate
#    (ZAP: Options → Network → Server Certificates → Save)

# 2. Converter DER -> PEM (Burp exporta cert.der por padrão)
openssl x509 -inform DER -in cacert.der -out burp.crt

# 3. Instalar no sistema (use o comando do seu SO)
sudo trust anchor --store burp.crt      # Arch/CachyOS (p11-kit)
sudo update-ca-certificates             # Debian/Kali (não existe no Arch)

# 4. Browser Firefox usa store próprio: Settings → Certificates → View → Import
```

```bash
# curl confiando no CA do proxy (sem -k)
curl --proxy http://127.0.0.1:8080 --cacert burp.crt https://<DOMINIO>/

# Aceitar certificado inválido (somente em teste/lab)
curl -k --proxy http://127.0.0.1:8080 https://<DOMINIO>/
```

**Quando usar:** sem o CA importado, todo HTTPS aparece como erro de certificado e o intercept não funciona.

## 🎯 Escopo (Target → Scope)

```text
Target → Scope → Add → https://<DOMINIO>
[✓] Include subdomains
Proxy → Options → [✓] And URL is in target scope   (limita o que é interceptado/gravado)

# ZAP: setas no Sites tree → Add to Context (context define o que o Scan ataca)
```

**Por quê:** o escopo restringe interceptação, histórico e varreduras ao alvo autorizado — evita tráfego fora de escopo no relatório.

## 🖱️ Intercept e Repeater

| Ação | Atalho |
|---|---|
| Intercept on/off | `Proxy → Intercept is on` |
| Forward / Drop | botões na aba Intercept |
| Enviar para Repeater | `Ctrl+R` |
| Enviar para Intruder | `Ctrl+I` |
| Enviar para Comparer | clique direito → Send to Comparer |
| Repetir requisição | `Ctrl+Space` no Repeater |

**Fluxo:** intercepte → `Ctrl+R` → desligue o intercept → modifique e reenvie no Repeater. Trabalhe sempre com intercept **desligado** e use o Repeater; interceptar cada requisição é lento.

```text
Repeater útil para: testar IDOR trocando id, manipular cookie/JWT, alterar method/headers,
testar bypass de autorização, ver a resposta bruta.
```

## 🔁 Intruder — 4 modos

| Modo | Comportamento | Uso típico |
|---|---|---|
| **Sniper** | 1 payload set, uma posição por vez | Fuzzing de um parâmetro; testar várias posições |
| **Battering ram** | Mesmo payload em todas as posições | Usuário=senha, repetir token |
| **Pitchfork** | Payload sets paralelos, casados por índice | Credenciais username/senha correspondentes de listas conhecidas |
| **Cluster bomb** | Produto cartesiano de todos os sets | Brute force usuário × senha, host × path |

```text
Preparação:
1. Ctrl+I → aba Intruder → Positions
2. Clear § → selecionar o valor → Add §  (posição marcada fica entre §)
3. Attack type → escolher
4. Payloads → load wordlist; Grep - Match para achar sinal de sucesso
5. Options → Request engine: thread pool, delay entre requisições (evitar rate limit)
6. Start attack → ordenar por Status/Length
```

```bash
# Equivalentes em linha de comando (mais rápidos e scriptáveis)
ffuf -u 'https://<DOMINIO>/api?id=FUZZ' -w ids.txt -mc 200 -fs 0
ffuf -u 'https://<DOMINIO>/login' -X POST -d 'user=admin&pass=FUZZ' \
     -H 'Content-Type: application/x-www-form-urlencoded' -w /usr/share/seclists/Passwords/Common-Credentials/10k-most-common.txt -fc 401
hydra -l admin -P passwords.txt <IP> http-post-form '/login:user=^USER^&pass=^PASS^:Invalid'
```

**Quando usar Intruder:** requisições com cookies, tokens CSRF e headers específicos que são dolorosos de montar no ffuf. Para brute force simples, prefira ffuf.

## 🧩 Decoder e Comparer

```text
Decoder: URL, Base64, Hex, Gzip, JWT. Use "Decode as" para encadear (ex.: Base64 → URL).
Comparer: selecione duas respostas → Words diff / Bytes diff → acha a diferença em respostas quase iguais
(ex.: um campo escondido que só aparece para admin).
```

## 🔧 Match and Replace

```text
Proxy → Options → Match and Replace → Add
```
| Tipo | Regex | Efeito |
|---|---|---|
| Request header | `^User-Agent:.*` | Força um User-Agent mobile/legado |
| Request header | `^Referer:.*` | Spoof de origem para contornar checagem lógica |
| Request body | `"role":"user"` | Escalada de privilégio em APIs JSON |
| Response header | `^Location:.*` | Regravar redirects |
| Response body | `admin` | Mascarar/alterar conteúdo para testar lógica |

**Quando usar:** quando a mesma modificação é necessária em dezenas de requisições — evita editar manualmente no Repeater.

## 🧪 Extensions úteis (BApp Store)

| Extensão | Para quê |
|---|---|
| Logger++ | Log avançado e filtrável de todo o tráfego |
| Param Miner | Descobre parâmetros/headers escondidos e cache poisoning |
| Turbo Intruder | Intruder em Python, com race conditions e alta concorrência |
| Autorize | Testa controle de acesso automaticamente (replay com outro usuário) |
| JSON Web Tokens | Editar/atacar JWT, testar `alg:none` e chaves fracas |
| Active Scan++ | Checks extras no scanner |
| Backslash Powered Scanner | Detecção heurística de injeções |
| Collaborator / `interactsh` | Interações OOB (SSRF, blind XSS); no Burp Pro é o Collaborator |

## 🕷️ OWASP ZAP — equivalente

| Burp | ZAP |
|---|---|
| Proxy → Intercept | Break (barra superior) |
| Repeater | Request/Response tabs (Manual Request Editor) |
| Intruder | Fuzzer (seleciona payloads no contexto) |
| Target → Scope | Contexts (menu de contexto na Sites tree) |
| Scanner (Pro) | Active Scan (livre na versão community) |
| Extensions (BApp) | Marketplace / Add-ons |
| Decoder | Encoder/Decoder/Hash |
| Spider | Spider + Ajax Spider (navegador headless real) |

```bash
# ZAP em modo daemon, com API para automação
zaproxy -daemon -host 127.0.0.1 -port 8080
# Baseline scan automatizado de um alvo
docker run -t ghcr.io/zaproxy/zaproxy:stable zap-baseline.py -t https://<DOMINIO> -r report.html
```

**Quando usar ZAP:** quando precisa de varredura ativa/spider sem licença Pro; `ajax spider` do ZAP é melhor que o Burp Community em aplicações SPA.

## 🖥️ Ferramentas de CLI via proxy

```bash
# curl através do proxy
curl --proxy http://127.0.0.1:8080 https://<DOMINIO>/ -v
curl --proxy http://<USUARIO>:<SENHA>@127.0.0.1:8080 https://<DOMINIO>/

# ffuf via Burp: cada requisição (inclusive os matches) aparece no histórico
ffuf -u https://<DOMINIO>/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
     -x http://127.0.0.1:8080 -replay-proxy http://127.0.0.1:8080

# sqlmap via Burp: aproveita cookies/sessão já capturados
sqlmap -u 'https://<DOMINIO>/item?id=1' --proxy=http://127.0.0.1:8080 --cookie='<COOKIE>'

# Variáveis de ambiente (vale para qualquer ferramenta que respeite)
export http_proxy=http://127.0.0.1:8080 https_proxy=http://127.0.0.1:8080
```

**Quando usar:** `-x` no ffuf para gravar o ataque inteiro no Burp e depois investigar manualmente os casos estranhos. `--replay-proxy` manda somente os hits (menos ruído).

## ⚠️ Pegadinhas

- **Porta 8080 em conflito**: Burp e ZAP não coexistem no mesmo listener.
- **Firefox ignora o CA do sistema**: importe o certificado no store próprio do Firefox.
- **`--cacert` vs `-k`**: `-k` desabilita a validação — ok no lab, ruim para entender erros de TLS reais.
- **Intruder Community é limitado** (threads e rate): use ffuf ou Turbo Intruder para volume.
- **Cluster bomb explode a lista**: 100 usuários × 100 senhas = 10.000 requisições; calcule antes.
- **Sniper em N posições** executa N × payloads, não payloads — leia o contador.
- **Match/replace global** pode quebrar outras requisições do histórico; deixa regra ativa só durante o teste.
- **Escopo mal configurado** faz você atacar host fora do escopo autorizado — valide `Target → Scope`.
- **ZAP Active Scan ataca de verdade**: pode derrubar a aplicação e gerar dados inválidos.
- **HSTS/certificate pinning** em apps mobile: proxy não intermedeia sem bypass adicional.

## ✅ Checklist rápido

- [ ] Proxy ouvindo em 127.0.0.1:8080 e browser apontando para ele (FoxyProxy por pattern)
- [ ] CA do proxy instalado no browser e/ou no sistema (`trust anchor --store` no Arch)
- [ ] Escopo do alvo configurado — sem tráfego de terceiros no histórico
- [ ] Intercept desligado por padrão; Repeater para testes manuais
- [ ] Intruder/ffuf escolhido conforme necessidade (posições vs volume)
- [ ] Match and Replace aplicado quando a alteração é repetitiva
- [ ] Extensões carregadas: Logger++, Param Miner, JWT, Turbo Intruder
- [ ] Ferramentas de CLI com `--proxy`/`-x` apontando para o proxy
- [ ] Requisições-chave salvas como evidência (raw request/response + timestamp)

## 🔗 Relacionado

- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[CPTS 05 - Information Gathering - Web Edition]]
- [[CPTS 23 - Web Attacks]]
- [[Fase 05 - Enumeração Web]]
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]
