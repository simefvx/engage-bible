---
tema: Pentest / Metodologia
tópico: Fase 01 — Pré-Engajamento
tags: [htb, cpts, engage, pre-engajamento, escopo, roe, contrato, setup]
dificuldade: fundamental
---

# 📝 Fase 01 — Pré-Engajamento

> [!info] Objetivo
> Transformar um pedido vago ("quero um pentest") em um trabalho **legal, delimitado e seguro**: escopo escrito, regras de engajamento (RoE), janela, contatos, tratamento de dados e ambiente de ataque pronto.
> Nada é testado nesta fase. O produto dela é **papel assinado + nota de engajamento criada + atacante pronto para o dia 1**.

---

## 🎯 Quando entrar

- Antes de qualquer pacote sair para a rede do cliente.
- Ao renovar contrato, mudar de escopo ou entrar em um teste herdado (mesmo que outra pessoa tenha começado).
- Sempre que o cliente pedir algo no meio do teste que **não** está no documento atual.

> [!warning] Sem autorização escrita e assinada não existe pentest — existe crime.
> Se surgir pedido fora do escopo durante a execução: **parar, registrar, formalizar aditivo**, só então continuar.

---

## 📥 Pré-requisitos e inputs

| Input | De quem vem | Para que serve |
|-------|-------------|----------------|
| Contrato / SOW | Comercial / jurídico | Base legal do teste e entregáveis |
| Escopo de alvos | Cliente | Lista normalizada de IPs, domínios, apps, contas |
| RoE assinado | Cliente + você | Limites técnicos e proibições |
| Janela de teste | Cliente | Quando pode rodar, horário de silêncio |
| Contatos de emergência | Cliente | Quem acionar em incidente, 24x7 |
| Credenciais (se houver) | Cliente | Define black/grey/white-box |
| Template de relatório | Você | Padroniza a saída |
| `client.ovpn` / acesso VPN | Cliente | Conectividade com o alvo |

---

## 🪜 Passo a passo

### 1. Definir o tipo de teste

| Tipo | Conhecimento inicial | Vantagem | Custo/tempo |
|------|----------------------|----------|-------------|
| **Black-box** | Nenhum. Você é atacante externo. | Simula atacante real; mede detecção. | Maior tempo, mais ruído |
| **Grey-box** | Credencial de usuário baixo privilégio ou parte da infra. | Cobre mais em menos tempo; foca em privesc/movimento lateral. | Equilíbrio |
| **White-box** | Código, diagramas, credenciais admin, tudo. | Máxima cobertura e profundidade de código. | Não mede detecção |

> [!tip] Combine o tipo com o objetivo do cliente
> Se o objetivo é "saber se nos detectam", black-box com janela acordada. Se é "corrigir o máximo antes do audit", white-box. Grey-box é o padrão da maioria dos engages de rede.

### 2. Normalizar o escopo

Liste e **valide** cada alvo. IP fora de faixa autorizada = incidente.

```bash
# Cria a estrutura de trabalho do engage (uma pasta por fase do teste)
mkdir -p ~/engages/<CLIENTE>/{00-admin,01-recon,02-scans,03-enum,04-exploit,05-loot,06-evidencias,07-relatorio}
# 00-admin guarda contrato, RoE, escopo, contatos e as notas do engage
```

Anote no escopo: faixas CIDR, domínios e **subdomínios** (incluem wildcard?), IPs de terceiros (cloud/CDN) — CDN do cliente é alvo ou não? Contas de teste fornecidas. Ambientes excluídos (produção, OT/ICS, backups).

### 3. Redigir/fechar as regras de engajamento (RoE)

Cláusulas que **devem** estar explícitas:

| Cláusula | Pergunta que responde |
|----------|----------------------|
| Alvos permitidos | Exatamente o que pode ser tocado (IP/porta/app/API) |
| Alvos proibidos | Terceiros, OT, backups, sistemas médicos/financeiros |
| Técnicas permitidas | Phishing, engenharia social, DoS/DDoS, força bruta, exploração destrutiva |
| Janela e silêncio | Dias, horários, blackout de produção |
| Ruído máximo | Rate limit acordado; evitar scan full-rate em horário comercial |
| Tratamento de dados | PII/dados sensíveis: não exfiltrar, mascarar, destruir ao fim |
| Retenção e destruição | Quando apagar loot, evidências e credenciais capturadas |
| Credenciais | Como armazenar (cofre), quem pode ver, rotação ao fim |
| Regras de parada | O que interrompe o teste (impacto em produção, incidente real) |
| Contatos | Autorizador, contato técnico, plantão de emergência |
| Legal/aviso | Autorização de teste, não-teste, escopo legal, seguro |

### 4. Definir janela, contatos e procedimento de emergência

- **Janela:** data/hora de início e fim, fuso, horário de silêncio, mudança de escopo.
- **Contatos:** nome, cargo, telefone/e-mail, canal (Signal? telefone?), disponibilidade.
- **Parada:** quem pode mandar parar, como você confirma a parada, qual o comando de rollback.
- **Acionamento:** se um ataque disparar alerta, o analista do cliente liga para você **antes** de tratar como incidente real.

> [!danger] Teste a comunicação ANTES do dia 1
> Troque uma mensagem de teste com o contato de emergência. Se o canal falhar no meio de um exploit em produção, você não tem plano B.

### 5. Tratar dados (PII, loot, evidências)

```bash
# Criptografa a pasta de loot do engage em repouso (recomendado para PII)
gpg --symmetric --cipher-algo AES256 ~/engages/<CLIENTE>/05-loot/creds.txt
# Trabalhe com dados mascarados nos relatórios: mostra 4 primeiros/últimos chars de PII
```

Regras práticas: não baixar base de dados inteira para "provar"; capturar amostra + esquema. Não mover PII para fora da rede do cliente se o RoE proibir. Registrar hash/data de tudo que foi coletado e quando foi destruído.

### 6. Preparar o atacante (setup)

```bash
# --- Conectividade ---
sudo openvpn --config ~/engages/<CLIENTE>/00-admin/client.ovpn   # sobe o túnel
ip a show tun0                                                   # confirma IP atribuído na VPN
sudo ip route get <IP-ALVO>                                      # confirma rota pelo tun0

# --- Wordlists (CachyOS/Arch) ---
sudo pacman -S --needed seclists                                # instala em /usr/share/seclists
ls -lh ~/wordlists/rockyou.txt                                  # já descompactado nesta máquina
ls /usr/share/seclists/Discovery/Web-Content/                   # common.txt, DirBuster-2007_directory-list-2.3-medium.txt
ls /usr/share/seclists/Usernames/                               # listas de usuários (nome.sobrenome etc.)

# --- Ferramentas base (pacman) ---
sudo pacman -S --needed nmap gobuster ffuf feroxbuster whatweb hydra john hashcat \
     smbclient smbmap enum4linux-ng dnsrecon dnsenum fierce smtp-user-enum \
     onesixtyone net-snmp snmpcheck ipmitool perl-image-exiftool seclists
sudo pacman -S --needed netexec impacket                        # nxc e secretsdump.py / smbserver.py / mssqlclient.py
sudo pacman -S --needed ssh-audit
pipx install cloud_enum                                         # não está nos repos oficiais
sudo pacman -S --needed bloodhound                              # BloodHound CE (coletor via SharpHound/Hound)

# --- Ferramentas complementares (não vieram no grupo base) ---
sudo pacman -S --needed openldap freerdp ipmitool swaks net-snmp \
     onesixtyone smtp-user-enum commix joomscan droopescan hakrawler xsstrike ligolo-ng
# odat (Oracle) não está nos repos oficiais: git clone https://github.com/quentinhardy/odat
# enum4linux clássico foi substituído por enum4linux-ng; crackmapexec por nxc (netexec)

# --- Ferramentas Go (ProjectDiscovery e afins) ---
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
go install -v github.com/lc/gau/v2/cmd/gau@latest
go install -v github.com/tomnomnom/waybackurls@latest
go install -v github.com/tomnomnom/assetfinder@latest
export PATH="$PATH:$(go env GOPATH)/bin"

# --- Proxy / ferramentas gráficas ---
burpsuite &                                                          # ou ZAP
```

Checklist do atacante:

- [ ] VPN conectada e rota confirmada ao alvo
- [ ] `/etc/hosts` recebe os hosts descobertos (`<IP> <DOMINIO>`)
- [ ] Wordlists instaladas (SecLists + rockyou + custom de nomes)
- [ ] nmap, ffuf/gobuster, netexec, impacket, responder, chisel/ligolo-ng
- [ ] Burp com CA instalada no proxy do navegador
- [ ] Terminal com scrollback grande; logging de sessão ligado
- [ ] Pasta `~/engages/<CLIENTE>` criada e sincronizada com backup

### 7. Criar a nota de engajamento (o "diário" do teste)

Template de anotação que deve existir **antes** do primeiro pacote:

```markdown
# Engajamento <CLIENTE> — <TIPO>
- Escopo: <IPs / DOMINIOS>
- Fora de escopo: <...>
- Janela: <inicio> a <fim>
- RoE: <proibido: DoS, phish, ...>
- Contatos: <nome> <fone> (emergencia: <nome> <fone>)
- Credenciais fornecidas: <...>

## Log (mais recente no topo)
- 2026-10-18 09:12 — 10.10.10.5 — nmap -sS top1000 — 22,80,445 abertos
- 2026-10-18 09:40 — 10.10.10.5:445 — null session permitida; shares: IT, Backups
```

> [!tip] Log com timestamp e comando, não com conclusão
> Escreva o comando e a saída bruta. "Achei um SMB estranho" três semanas depois não vale nada; a linha de saída com hora vale tudo.

### 8. Perguntas que devem ser respondidas ANTES de começar

1. Quais alvos estão **dentro** e quais estão **fora**?
2. Existem terceiros (cloud, CDN, SaaS) no caminho? Podem ser tocados?
3. Posso usar phishing, brute force ou qualquer técnica que cause indisponibilidade?
4. Qual o horário de silêncio e a janela exata?
5. Quem é o autorizador e quem é o plantão de emergência?
6. Qual o critério de "parada imediata"?
7. É black, grey ou white-box? Tenho credenciais?
8. Como os dados sensíveis encontrados devem ser tratados e destruídos?
9. Qual o formato do entregável, prazo e haverá reteste?
10. Há sistema de monitoramento/EDR? O cliente espera alertas de quê?

---

## 🧰 Ferramentas

| Ferramenta | Uso no pré-engajamento |
|------------|------------------------|
| `openvpn` / `wireguard` | Conectar à rede do cliente |
| `obsidian` | Manter a nota do engage e a base de conhecimento |
| `git` (repo privado) | Versionar comandos, scripts e templates de relatório |
| `gpg` | Criptografar loot e evidências em repouso |
| `seclists` | Wordlists padronizadas |
| `tmux` | Sessões persistentes durante o teste |
| `timewarrior` / planilha | Timetracking e evidência de horas no contrato |
| Template de relatório | Saída padronizada (ver CPTS 27) |

---

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| Cliente não assinou autorização escrita | Não começar. Emitir documento e cobrar assinatura. |
| Escopo ambíguo (faixa sem máscara, "todos os sites") | Bloquear início; pedir lista explícita por escrito. |
| RoE proíbe DoS, mas você precisa testar falha que derruba serviço | Testar só com autorização pontual e em janela; documentar. |
| Cliente forneceu credencial de domínio admin | Reclassificar para white-box e ajustar metodologia/entregável. |
| Alvo resolvido aponta para CDN de terceiro | Confirmar autorização do provedor; senão, excluir do escopo. |
| Iceberg: pedido verbal de novo alvo no meio do teste | Parar, pedir aditivo escrito, registrar no log. |
| Descoberta de vazamento de dado sensível real | Parar, notificar contato de emergência, aguardar instrução. |
| Sem VPN / rota errada | Corrigir conectividade antes de qualquer scan (`ip route get`). |

---

## 📤 Evidências a registrar

- Contrato/SOW e **RoE assinado** (PDF) em `00-admin/`.
- Escopo normalizado (CSV/MD) com IPs, domínios, contas.
- Janela, contatos e procedimento de emergência.
- Credenciais fornecidas (no cofre, **fora** do relatório).
- Print/registro da VPN conectada com IP atribuído e data.
- Versão das ferramentas usadas (para reprodutibilidade do relatório).
- Nota de engajamento criada e com o primeiro log de conectividade.

---

## 🕳️ Pegadinhas

- 🕳️ **Escopo por planilha desatualizada.** Confirme sempre a versão mais recente com o cliente.
- 🕳️ **Assumir que wildcard `*.dominio` cobre ipv6/cloud.** Não cobre; confirme.
- 🕳️ **Trabalhar em produção sem janela acordada.** Um `--min-rate` alto derruba firewall pequeno.
- 🕳️ **Guardar credenciais no relatório ou em texto plano.** Cofre + entrega fora do entregável.
- 🕳️ **Não anotar a hora.** Toda evidência sem timestamp perde valor jurídico e técnico.
- 🕳️ **Esquecer o `client.ovpn` renovado.** Túnel cai no meio do teste.
- 🕳️ **Falar com o "chefe" no meio do teste e aceitar ordem verbal.** Só o autorizador formal conta.
- 🕳️ **Confundir "escanear" com "explorar".** O RoE pode permitir um e proibir o outro.

---

## ✅ Checklist

- [ ] Autorização escrita assinada e arquivada
- [ ] RoE com todas as cláusulas da tabela preenchidas
- [ ] Escopo normalizado e validado (dentro × fora)
- [ ] Janela, horário de silêncio e fuso confirmados
- [ ] Contatos de emergência testados
- [ ] Plano de tratamento e destruição de dados definido
- [ ] VPN conectada e rota ao alvo validada
- [ ] Estrutura de pastas do engage criada
- [ ] Wordlists e ferramentas instaladas e testadas
- [ ] Nota de engajamento criada com template de log
- [ ] Tipo de teste (black/grey/white) definido
- [ ] Perguntas da seção 8 respondidas por escrito
- [ ] Template de relatório escolhido
- [ ] Cronograma acordado e entregáveis claros

---

## 🔗 Relacionado

- [[Workflow do Engage]]
- [[CPTS 01 - Penetration Testing Process]]
- [[CPTS 27 - Documentation and Reporting]]
- [[Fase 02 - OSINT e Reconhecimento Externo]]
- [[Fase 12 - Relatório e Encerramento]]
