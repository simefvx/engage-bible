---
tema: Pentest / Reconhecimento
tópico: Fase 03 — Reconhecimento de Rede com Nmap
tags: [htb, cpts, engage, nmap, recon, scan, portas, nse]
dificuldade: easy
---

# 🛰️ Fase 03 — Reconhecimento de Rede

> [!info] Objetivo
> Descobrir **quais hosts existem** e **quais portas respondem** na faixa do escopo, com versão de serviço onde fizer sentido — sempre salvo em arquivo (`-oA`), sempre datado, sempre dentro da janela.
> Saída: `scans/<ip>/` com `.nmap/.xml/.gnmap` + tabela host × porta × serviço × versão × evidência.

---

## 🎯 Quando entrar

- Depois do recon externo (Fase 02) entregar a lista de IPs/faixas dentro do escopo.
- Ao ganhar acesso a uma nova rede interna (via pivoting) — a rede interna tem alvos que a externa não mostra.
- Após cada nova credencial, para reavaliar o que agora é legível.

> [!warning] Scan grande é o evento mais visível do engage
> `-p-` em `/16` com `--min-rate 10000` derruba firewall pequeno e acende IDS. Confirme a janela e comece conservador.

---

## 📥 Pré-requisitos e inputs

| Input | Origem | Uso |
|-------|--------|-----|
| Lista de IPs/CIDR no escopo | Fase 01/02 | Alvo do scan |
| Janela e regras de ruído | RoE | Limita `-T` e `--min-rate` |
| VPN conectada e rota válida | Setup | Sem isso, os pacotes não chegam |
| `/etc/hosts` preparado | Você | Nomes resolvíveis durante a enum |
| Pasta `scans/` criada | Você | Todo scan sempre salvo |
| Credencial/proxy (se scan interno) | Pivoting | Scan via proxychains |

```bash
mkdir -p ~/engages/<CLIENTE>/02-scans && cd ~/engages/<CLIENTE>/02-scans
ip route get <IP>          # confirma que o tráfego sai pelo tun0, não pela internet
```

---

## 🪜 Passo a passo

### 1. Descoberta de hosts

```bash
# Ping scan simples (ICMP echo + TCP) numa faixa — lista hosts vivos, sem portas
nmap -sn 10.10.10.0/24 -oA discovery_cidr

# ARP scan quando o alvo é a mesma LAN (mais confiável, ignora firewall de ICMP)
sudo nmap -sn -PR 10.10.10.0/24

# Quando ICMP é bloqueado: sonda TCP em portas que costumam estar abertas
nmap -sn -PS22,80,443,445,3389 -PA80,443 10.10.10.0/24
```

Se nem `-sn` responde, **não desista**: passe direto para scan de portas com `-Pn` (pula a descoberta e trata todo host como vivo).

```bash
# Trata todos como vivos: usa quando o cliente bloqueia ICMP/ping
nmap -Pn -sS --top-ports 100 --open -oA discovery_pn 10.10.10.0/24
```

### 2. Scan rápido das portas mais comuns (top 1000)

```bash
# SYN scan das 1000 portas mais comuns, sem DNS reverso (-n), só abertas, rápido
sudo nmap -sS -Pn -n --open --top-ports 1000 -T4 --min-rate 1000 \
  -oA top1000_<IP> <IP>
# --top-ports 1000  as mil portas mais frequentes (mais rápida que -p-)
# -sS  SYN scan (meio-aberto, rápido, requer root)
# -Pn  não faz ping — assume host vivo
# -T4  timing agressivo (ok em rede de laboratório/interna)
# --min-rate  pacotes por segundo mínimos (não esperar timeout)
```

### 3. Scan completo de todas as portas

Top 1000 **sempre** esconde serviço. Faça `-p-` em todo host vivo.

```bash
# Todas as 65535 portas TCP: rápido, só portas abertas, retries baixos
sudo nmap -p- -Pn -n --open --min-rate 5000 --max-retries 1 -T4 \
  -oA allports_<IP> <IP>

# Alternativa em duas etapas (menos CPU/memória em faixas grandes)
sudo nmap -p- -Pn -n --min-rate 3000 --open -oG - <IP> | grep -oP '\d+(?=/open)' | paste -sd,
```

> [!tip] Faixa inteira em uma passada
> `nmap -p- -Pn -n --open --min-rate 5000 -T4 -iL hosts.txt -oA allports_all` roda todos os hosts em paralelo. Um `-oA` só, e você já tem o inventário completo.

### 4. Detecção de serviço e scripts default

Rode **depois** de saber as portas abertas — assim é rápido e cirúrgico:

```bash
# Versão + scripts default nas portas que o -p- achou
sudo nmap -sC -sV -p 22,80,139,445,3306 -Pn -n -oA services_<IP> <IP>
# -sC  roda os scripts da categoria "default" (banners, títulos, shares)
# -sV  identifica produto/versão do serviço

# Mais agressivo na identificação de versão quando o banner é enganoso
sudo nmap -sV --version-all -p <PORTAS> -oA version_all_<IP> <IP>

# Só banner (rápido, para triagem)
sudo nmap -sV --script=banner -p <PORTAS> <IP>
```

### 5. Varredura UDP

UDP é lento, então priorize as portas que interessam:

```bash
# Top 100 UDP (DNS, SNMP, NetBIOS, NTP, IKE, TFTP, IPMI...)
sudo nmap -sU --top-ports 100 -Pn -n --open -oA udp_top100_<IP> <IP>

# Portas UDP de alto valor explicitamente
sudo nmap -sU -p 53,67,69,111,123,137,161,162,500,1900,5353 -Pn --open -oA udp_high_<IP> <IP>
```

| Porta UDP | Serviço | Por que importa |
|-----------|---------|-----------------|
| 53 | DNS | AXFR, subdomínios, cache |
| 69 | TFTP | Arquivos de config sem auth |
| 111 | rpcbind | Enumera NFS/RPC |
| 123 | NTP | `monlist`, recon de rede |
| 137/138 | NetBIOS | Nomes, domínios |
| 161/162 | SNMP | Community string, OIDs, processos |
| 500 | IKE | VPN, PSK |

### 6. NSE por serviço

Depois de saber o que roda, use scripts específicos:

```bash
# SMB: shares, usuários, SO, versão e vulnerabilidades conhecidas
sudo nmap -p 139,445 --script smb-os-discovery,smb-enum-shares,smb-enum-users,smb2-security-mode \
  -oA nse_smb_<IP> <IP>
sudo nmap -p 445 --script "smb-vuln-*" --script-args=unsafe=1 <IP>   # cuidado: intrusive

# HTTP: título, headers, métodos, diretórios comuns, robots
sudo nmap -p 80,443 --script http-title,http-headers,http-methods,http-enum,http-robots.txt <IP>

# FTP: anonymous, bounce, syst
sudo nmap -p 21 --script ftp-anon,ftp-bounce,ftp-syst,ftp-proftpd-backdoor <IP>

# DNS e SNMP
sudo nmap -p 53 --script dns-zone-transfer,dns-brute --script-args dns-brute.domain=<DOMINIO> <IP>
sudo nmap -sU -p 161 --script snmp-info,snmp-sysdescr,snmp-interfaces,snmp-processes <IP>
```

Categorias NSE: `auth`, `broadcast`, `brute`, `default`, `discovery`, `dos`, `exploit`, `external`, `fuzzer`, `intrusive`, `malware`, `safe`, `version`, `vuln`.

```bash
# Ver quais scripts existem para um serviço/categoria
ls /usr/share/nmap/scripts/ | grep smb-
nmap --script-help "smb-vuln-*"
sudo nmap --script-updatedb      # atualiza o banco de scripts
```

### 7. Timing: escolher a velocidade certa

| Template | Nome | Quando usar |
|----------|------|-------------|
| `-T0` | Paranoid | IDS agressivo; 1 porta a cada 5 min |
| `-T1` | Sneaky | Evasão; muito lento |
| `-T2` | Polite | Produção, horário comercial |
| `-T3` | Normal | **Padrão**; rede com perda de pacote |
| `-T4` | Aggressive | Lab / rede confiável (o mais usado) |
| `-T5` | Insane | Só em CTF/rede local; perde exatidão |

```bash
# Ajuste fino em vez de -T5
sudo nmap -p- --min-rate 5000 --max-retries 1 --host-timeout 15m --scan-delay 5ms <IP>
sudo nmap -p- --max-rate 100 -T2 <IP>          # produção: teto de 100 pkt/s
nmap -v --stats-every 10s -p- <IP>             # progresso e ETA
```

### 8. Estados de porta (leia o resultado corretamente)

| Estado | Significado | O que fazer |
|--------|-------------|-------------|
| `open` | Serviço aceitando conexão | Enumerar (Fase 04) |
| `closed` | Host vivo, porta sem serviço | Confirma que o host existe |
| `filtered` | Firewall descartou a sonda | Testar outra técnica/porta; suspeitar de firewall |
| `unfiltered` | Alcançável, mas ACK scan não diz aberta/fechada | Complementar com `-sS`/`-sV` |
| `open\|filtered` | Sem resposta (comum em UDP) | Repetir com `-sU` e script específico |
| `closed\|filtered` | Só idle scan retorna isso | Ambíguo — tratar como fechado |

```bash
# --reason diz POR QUE o nmap concluiu cada estado (evidência para o relatório)
sudo nmap -p 22,80 --reason -oA reason_<IP> <IP>
```

### 9. Salvar e parsear a saída

```bash
# Sempre os três formatos: normal (leitura), XML (parsing), greppable (grep)
sudo nmap -sC -sV -p- -oA scans/<IP>_full <IP>

# Converte o XML em HTML legível para o relatório
xsltproc -o <IP>.html /usr/share/nmap/nmap.xsl scans/<IP>_full.xml

# Extrai portas abertas do greppable (vira lista para a próxima fase)
grep -oP '\d+(?=/open)' scans/<IP>_full.gnmap | sort -un | paste -sd,
# Lista "IP:porta" para passar a outras ferramentas
awk '/Ports:/{split($0,a,"Ports: ");n=split(a[2],p,", ");for(i=1;i<=n;i++){split(p[i],q,"/");if(q[2]=="open")print $2":"q[1]}}' scans/<IP>_full.gnmap

# Tabela consolidada de todos os hosts
grep -h 'Ports:' scans/*.gnmap | sed 's/Host: //; s/ ()//' 
```

### 10. Scan de rede interna via proxy

Quando o alvo só é alcançável através de um pivô:

```bash
# proxychains: TCP connect scan (-sT) porque SYN raw não passa pelo SOCKS
# -Pn obrigatório (ICMP/ping não atravessa o proxy)
proxychains4 -q nmap -sT -Pn -n --open -p 22,80,139,445,3389,5985 10.10.20.0/24 -oA internal_sw

# Descobrir hosts vivos pela rede interna (sem ICMP disponível)
proxychains4 -q nmap -sT -Pn -n --open -p 445,3389,80 10.10.20.0/24
```

> [!tip] Não rode `-sS` nem `-O` por proxy
> O proxy só entende connect() TCP. SYN scan, OS detection e UDP não funcionam através dele.

### 11. Cuidados com IDS/firewall

```bash
# Fragmentação + MTU pequeno: quebra a assinatura do pacote (MTU múltiplo de 8)
sudo nmap -f --mtu 8 -p- <IP>

# Desvios: mistura scans com IPs isca (RND:N gera N iscas aleatórias)
sudo nmap -D RND:10 -p- <IP>

# Origem/porta de origem "confiável" + dados falsos para mudar o tamanho do pacote
sudo nmap --source-port 53 --data-length 50 -p- <IP>

# Idle scan (zumbi): o scan parece vir de outra máquina (exige zumbi ocioso)
sudo nmap -sI <ZUMBI> -p- <IP>

# Embaralhar ordem dos hosts para não parecer varredura sequencial
sudo nmap --randomize-hosts -iL hosts.txt -p 80,443
```

> [!danger] Evasão muda a confiabilidade do resultado
> Fragmentação e iscas aumentam falso positivo/negativo. Sempre valide a porta crítica com um teste simples (`-sT`, `nc`) antes de concluir.

### 12. Consolidar

Monte o inventário: `host | portas abertas | serviço/versão | SO provável | observação`. É o input direto da Fase 04.

---

## 🧰 Ferramentas

| Ferramenta | Função | Observação |
|------------|--------|------------|
| `nmap` | Scanner principal (TCP/UDP, NSE) | Obrigatório em toda rede |
| `masscan` | Scan de faixa inteira em segundos | Use para triagem; depois detalhe com nmap |
| `rustscan` | Wrapper rápido que chama nmap | Bom para `-p-` rápido |
| `naabu` | Scan de portas rápido (ProjectDiscovery) | Integra com pipelines |
| `nmap-parse-output` | Parseia XML do nmap | Tabelas e listas prontas |
| `xsltproc` | XML → HTML para relatório | Nativo |
| `arp-scan` | Descoberta em LAN /24 | Muito confiável no segmento local |
| `fping` | Ping rápido em massa | Descoberta inicial |
| `proxychains4` | Scan/trafego pelo pivô | Só TCP connect |

---

## 🔀 Árvore de decisão

| Condição | Ação |
|----------|------|
| `-sn` retorna 0 hosts | Assumir ICMP bloqueado; usar `-Pn` + top portas |
| Faixa grande (`/16`) e janela curta | Triagem com `masscan`/`naabu`, depois `-sC -sV` só nos vivos |
| Portas top 1000 fechadas/filtered | Rodar `-p-` antes de concluir que o host está morto |
| Porta aparece como `filtered` | Suspeitar de firewall; tentar `--source-port 53`, `-f`, ou outra porta de origem |
| Serviço com banner enganoso | `-sV --version-all` e NSE `banner`; conferir manualmente com `nc` |
| Só HTTP/HTTPS respondendo | Ir direto para Fase 05 (web) — costuma ser o caminho mais curto |
| Host só alcançável por pivô | `proxychains4 nmap -sT -Pn`; nunca `-sS`/`-O`/`-sU` |
| IDS confirmado pelo cliente | Reduzir para `-T2`, `--max-rate`, escalonar por horário |
| Scan revela serviço de infra (DNS/AD/DB) | Priorizar enumeração em Fase 04/09 |
| Nmap lento/travando em `/16` | Dividir a faixa, usar `--host-timeout`, rodar em paralelo por bloco |

---

## 📤 Evidências a registrar

- Todos os arquivos `.nmap`, `.xml`, `.gnmap` por host em `02-scans/`.
- Data/hora e **comando completo** de cada execução (o `-oA` guarda o comando no header do `.nmap`).
- Tabela consolidada host × porta × serviço × versão.
- Estados de porta relevantes (`filtered`) e o `--reason` correspondente.
- Evidência de firewall/IDS ou de ICMP bloqueado.
- Inventário de portas UDP abertas.
- Resultado dos NSE scripts usados (especialmente `smb-os-discovery`, `http-title`).
- Interface/IP de origem usados (para provar que o tráfego saiu pelo túnel).

---

## 🕳️ Pegadinhas

- 🕳️ **Confiar só no top 1000.** O foothold costuma estar em porta alta (8080, 8443, 5985, 9001, 1433).
- 🕳️ **`-T5` em rede real.** Perde pacotes, marca portas como filtradas, gera falso negativo.
- 🕳️ **Não salvar com `-oA`.** Scan repetido é tempo perdido e ruído dobrado.
- 🕳️ **Esquecer UDP.** SNMP em 161 e TFTP em 69 entregam dados que TCP nunca mostrará.
- 🕳️ **Rodar `--script vuln` cedo.** É intrusive, pode derrubar serviço e ainda dá falso positivo.
- 🕳️ **Interpretar `filtered` como fechado.** Firewall com regra "deny" é diferente de serviço inexistente.
- 🕳️ **`-sS` através de proxychains.** Não funciona; use `-sT`.
- 🕳️ **Misturar IPs de terceiros (CDN) na faixa.** Escaneie só o que está autorizado.
- 🕳️ **Escanear sem VPN estável.** Resultado parcial silencioso — portas "faltando" que na verdade foram perdidas.
- 🕳️ **Varredura sequencial em horário comercial.** Padrão óbvio de scan para qualquer SIEM.

---

## ✅ Checklist

- [ ] Faixa de escopo conferida (sem IP de terceiro)
- [ ] VPN e rota validadas antes do primeiro pacote
- [ ] Descoberta de hosts (`-sn`, com fallback `-Pn`)
- [ ] Top 1000 portas em todos os hosts vivos
- [ ] `-p-` completo em todos os hosts vivos
- [ ] `-sC -sV` nas portas abertas
- [ ] UDP top-100 + portas de alto valor
- [ ] NSE por serviço executado
- [ ] Tudo salvo com `-oA` e comando registrado
- [ ] Saída parseada em tabela consolidada
- [ ] Scan interno via proxy (se houver pivô)
- [ ] Regras de ruído do RoE respeitadas
- [ ] Portas críticas revalidadas com teste simples
- [ ] Inventário pronto para a Fase 04

---

## 🔗 Relacionado

- [[Workflow do Engage]]
- [[CPTS 03 - Network Enumeration with Nmap]]
- [[Fase 02 - OSINT e Reconhecimento Externo]]
- [[Fase 04 - Enumeração de Serviços]]
- [[Fase 08 - Pivoting e Tunneling]]
