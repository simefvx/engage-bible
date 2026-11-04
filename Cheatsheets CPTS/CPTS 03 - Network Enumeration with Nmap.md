---
tema: Pentest / Reconhecimento
tópico: Network Enumeration with Nmap
tags: [htb, cpts, recon, nmap, scan, nse, tcp, udp, evasao]
dificuldade: easy
---

# CPTS 03 - Network Enumeration with Nmap — Cheatsheet

> [!info] Escopo deste cartão
> Cartão de referência do Nmap: descoberta de host, estados de porta, técnicas TCP/UDP, detecção de serviço/SO, motor NSE, performance, evasão de firewall/IDS e formatos de saída.
> Baseado no PDF oficial do módulo. Este é o cartão que você deixa aberto durante o scan.

---

## 🎯 Quando usar

| Situação | O que usar |
|----------|-----------|
| Rede nova, não sei quais hosts existem | Descoberta de host (`-sn`, `-Pn`) |
| Host conhecido, não sei as portas | Scan rápido + scan completo (`-p-`) |
| Quero versão de cada serviço | `-sV` |
| Quero banner e checagens básicas | `-sC` |
| Suspeito de serviço em UDP | `-sU --top-ports` |
| Preciso de vulnerabilidade conhecida | NSE (`--script vuln`) |
| Scan lento travando | Performance (`-T4`, `--min-rate`, RTT timeouts) |
| Firewall/IDS atrapalhando | Seção de evasão |
| Preciso do resultado para relatório | `-oA` + conversão XSLT |

---

## 🧰 Ferramentas

| Ferramenta | Função |
|------------|--------|
| `nmap` | Scanner principal |
| `nmap --script-updatedb` | Sincroniza a base de scripts NSE |
| `xsltproc` | Converte saída XML em HTML navegável |
| `ncat` | Netcat do projeto Nmap (versão mais completa) |
| `zenmap` | GUI do Nmap (opcional) |
| `/usr/share/nmap/scripts/` | Scripts NSE instalados |
| `/usr/share/nmap/nse_main.lua` | Motor NSE |

---

## 🖥️ Descoberta de host

| Opção | O que faz | Quando usar |
|-------|-----------|-------------|
| `-sn` | Só descobre host, **sem** scan de porta | Mapear rede antes de escanear |
| `-Pn` | Desliga ICMP Echo — assume host vivo | Alvo bloqueia ping |
| `-n` | Não resolve DNS | Ganhar tempo em rede grande |
| `-PE` | Ping via ICMP Echo Request | Descoberta clássica |
| `-PS<porta>` | TCP SYN ping na porta (padrão 80) | Firewall bloqueia ICMP |
| `-PA<porta>` | TCP ACK ping (padrão 80) | Alternativa ao SYN |
| `-PU<porta>` | UDP ping (padrão 40125) | Hosts que só respondem UDP |
| `-PR` | ARP ping (só na rede local) | Mais confiável em LAN |
| `--disable-arp-ping` | Desliga ARP ping | Força descoberta por IP |
| `--dns-server <ns>` | Usa um DNS específico | Resolver nomes internos |
| `--packet-trace` | Mostra cada pacote enviado/recebido | Debug de "por que não responde" |
| `--reason` | Mostra o motivo de cada resultado | Explicar estado no relatório |

```bash
# Descobre hosts vivos em uma faixa, sem escanear portas.
# O que faz: envia probes de descoberta e lista quem responde.
nmap -sn 10.10.10.0/24

# Mesma faixa sem ICMP, assumindo tudo vivo e só resolvendo o essencial.
nmap -sn -Pn -n 10.10.10.0/24

# Debug: quero ver o pacote que sai e a razão do estado de cada porta.
nmap -p80 --packet-trace --reason <IP>
```

---

## 🚪 Estados de porta

| Estado | Significado |
|--------|-------------|
| `open` | Serviço aceitando conexão |
| `closed` | Porta responde, sem serviço escutando |
| `filtered` | Firewall/ACL descartou — sem resposta |
| `unfiltered` | Responde, mas Nmap não sabe se está aberta (típico de ACK scan) |
| `open\|filtered` | Sem resposta possível: aberta (silenciosa) ou filtrada |
| `closed\|filtered` | Não dá para distinguir fechada de filtrada |

> 💡 `filtered` indica política de firewall. Registrar isso já é achado de reconhecimento.

---

## 🔍 Opções de scan

### Alvo e portas

| Opção | Descrição |
|-------|-----------|
| `10.10.10.0/24` | Faixa de rede alvo |
| `<IP> <IP> <IP>` | Vários alvos por espaço |
| `-iL hosts.txt` | Lista de alvos em arquivo |
| `-p-` | Todas as 65535 portas |
| `-p22,25` | Somente as portas listadas |
| `-p22-110` | Intervalo de portas |
| `-F` | Top 100 portas |
| `--top-ports=<num>` | Top N portas mais frequentes |
| `--exclude <IP>` | Exclui alvos do scan |

### Técnicas

| Opção | Descrição |
|-------|-----------|
| `-sS` | TCP SYN scan ("half-open", padrão com root) — rápido e discreto |
| `-sT` | TCP connect scan — usa a pilha do SO, sem root |
| `-sA` | TCP ACK scan — mapeia regras de firewall |
| `-sU` | UDP scan — lento, use `--top-ports` |
| `-sV` | Detecção de versão dos serviços |
| `-sC` | Script scan com scripts `default` (= `--script=default`) |
| `-O` | Detecção de sistema operacional |
| `-A` | `-O` + `-sV` + traceroute + scripts default |
| `-sI <zumbi>` | Idle scan — usa host zumbi, esconde o atacante |
| `--traceroute` | Traça a rota até o alvo |
| `-Pn` | Pula descoberta de host |
| `--scanflags` | Define flags TCP customizadas (ex.: `SYNACK`) |

```bash
# Passe rápido: top 1000 portas, versão e scripts default.
nmap -sV -sC <IP>

# Scan completo TCP: todas as portas, sem DNS, agressivo mas controlado.
nmap -p- --min-rate 1000 -T4 -Pn -n <IP>

# Descobre o que o firewall filtra comparando ACK scan com SYN scan.
sudo nmap -sA -p1-1024 <IP>

# UDP dos serviços mais prováveis (rápido o suficiente para ser útil).
sudo nmap -sU --top-ports 50 --open <IP>

# Detecção agressiva: SO, versão, traceroute, scripts default.
sudo nmap -A -p- <IP>
```

---

## 🧩 NSE — Nmap Scripting Engine

| Opção | Descrição |
|-------|-----------|
| `--script <script>` | Roda scripts específicos ou expressões |
| `--script "not intrusive"` | Filtra por categoria (linguagem NSE) |
| `--script "default or safe"` | Categorias combinadas com `or` |
| `--script "default and safe"` | Interseção de categorias |
| `--script "http-*"` | Wildcard por nome |
| `--script-args k=v` | Passa argumentos ao script |
| `--script-help <script>` | Mostra descrição e args do script |
| `--script-updatedb` | Atualiza o banco de scripts |
| `-sC` | Atalho para `--script=default` |

**Categorias NSE**

| Categoria | Significado |
|-----------|-------------|
| `default` | Rodadas por `-sC`; seguras e úteis |
| `safe` | Não derruba nem sobrecarrega o alvo |
| `intrusive` | Podem impactar o serviço ou ser detectadas |
| `auth` | Bypass/descoberta de autenticação |
| `brute` | Força bruta de credenciais |
| `discovery` | Descobre mais serviços e informações |
| `exploit` | Tenta explorar a falha |
| `vuln` | Detecta vulnerabilidades conhecidas |
| `dos` | Pode causar negação de serviço (cuidado!) |
| `fuzzer` | Envia dados malformados |
| `malware` | Procura backdoors/infecção |
| `broadcast` | Descoberta em rede local via broadcast |
| `external` | Consulta serviços de terceiros (ex.: whois) |
| `version` | Extensão da detecção de versão |

```bash
# Banner de serviços + info básica: bom primeiro contato com o alvo.
nmap -sV --script banner <IP>

# Só scripts não intrusivos em todas as portas abertas.
nmap -p- --script "default or safe" <IP>

# Checagens de vulnerabilidade em portas web e SMB.
nmap -p80,443,445 --script vuln <IP>

# Enumeração de web e SMB com scripts específicos.
nmap -p80,443 --script http-enum,http-headers,http-title <IP>
nmap -p445 --script smb-os-discovery,smb-enum-shares,smb-vuln-* <IP>

# Ver o que o script faz e quais argumentos aceita, antes de rodar.
nmap --script-help smb-os-discovery
```

> [!warning] Cuidado com `vuln` e `dos`
> `--script vuln` pode ser intrusivo. `dos` pode **derrubar** o serviço — só use com autorização explícita. `brute` gera muitos logins e pode bloquear conta.

---

## ⚡ Performance

| Opção | Descrição |
|-------|-----------|
| `--max-retries <num>` | Retentativas por porta |
| `--min-rate <num>` | Mínimo de pacotes por segundo |
| `--max-rate <num>` | Teto de pacotes por segundo |
| `--initial-rtt-timeout 50ms` | RTT inicial estimado |
| `--max-rtt-timeout 100ms` | RTT máximo antes de desistir |
| `--min-parallelism <num>` | Mínimo de probes em paralelo |
| `--max-parallelism <num>` | Teto de probes em paralelo |
| `--stats-every=5s` | Mostra progresso a cada 5s |
| `-v` / `-vv` | Verbosidade |
| `-T<0-5>` | Template de timing |

**Templates de timing**

| Template | Nome | Uso |
|----------|------|-----|
| `-T0` | paranoid | IDS evasão extrema (minutos entre probes) |
| `-T1` | sneaky | Silencioso, muito lento |
| `-T2` | polite | Baixa banda, respeita o alvo |
| `-T3` | normal | Padrão do Nmap |
| `-T4` | aggressive | Labs e redes rápidas — o mais usado |
| `-T5` | insane | Rede local/CTF; pode perder precisão |

```bash
# Rede rápida, evita espera longa por porta filtrada.
nmap -p- --min-rate 1000 --max-rtt-timeout 100ms -T4 <IP>

# Host lento/satélite: mais tolerância e retentativas.
nmap -sV -T2 --min-parallelism 10 --max-retries 5 <IP>

# Acompanhar progresso de um scan longo.
nmap -p- --stats-every=10s -v <IP>
```

---

## 🥷 Evasão de IDS/Firewall

| Opção | O que faz |
|-------|-----------|
| `-D RND:5` | 5 decoys aleatórios escondem seu IP no ruído |
| `-D decoy1,decoy2,ME` | Decoys explícitos; `ME` marca sua posição |
| `--spoof-mac <mac>` | Falsifica o MAC de origem (útil em LAN) |
| `--data-length <num>` | Adiciona bytes aleatórios ao pacote (muda a assinatura) |
| `-f` | Fragmenta os pacotes (fragmentação IP) |
| `--mtu <num>` | Define MTU customizado (múltiplo de 8) |
| `-S <IP>` | Falsifica IP de origem |
| `-e <iface>` | Escolhe a interface de saída |
| `-g <porta>` / `--source-port <porta>` | Falsifica a porta de origem (ex.: 53) |
| `--randomize-hosts` | Randomiza a ordem dos alvos |
| `--proxies <url>` | Roteia por proxies HTTP/SOCKS4 |

```bash
# Scan fragmentado, com decoys, MAC falso e pacotes maiores.
# Quebra assinatura de IDS que casa padrão de pacote típico do Nmap.
sudo nmap -p- -f --data-length 200 -D RND:10 --spoof-mac 0 -T2 <IP>

# Scan pela porta de origem 53 (parece tráfego DNS e fura ACL ingênua).
sudo nmap -p- -g 53 <IP>

# Interface específica + IP de origem específico (ambiente com múltiplas NICs).
sudo nmap -e tun0 -S <SEU_IP> -p445 <IP>
```

---

## 💾 Saída

| Opção | Formato | Uso |
|-------|---------|-----|
| `-oA <nome>` | Todos os 3 formatos | **Sempre use** — permite reprocessar |
| `-oN <nome>` | Normal (texto legível) | Ler e colar no relatório |
| `-oG <nome>` | Grepable | `grep` de portas abertas |
| `-oX <nome>` | XML | Consumido por ferramentas e conversão |
| `--reason` | — | Explica o estado de cada porta |
| `--open` | — | Mostra só portas abertas |

```bash
# Salva o scan completo em all/normal/grepable/XML.
nmap -sV -sC -p- -oA scans/nmap_full <IP>

# Extrai portas abertas da saída grepable.
grep -E "Ports:" scans/nmap_full.gnmap

# Converte XML em HTML para anexar ao relatório.
xsltproc scans/nmap_full.xml -o scans/nmap_full.html
```

---

## 🧪 Workflow de scan sugerido

```bash
# 1) Descobrir quais hosts existem na faixa.
nmap -sn -oA scans/discovery 10.10.10.0/24

# 2) Passe rápido para ter resultado em segundos.
nmap -sV -sC -oA scans/quick <IP>

# 3) Scan completo de portas, para não perder serviço em porta alta.
nmap -p- --min-rate 1000 -T4 -Pn -n -oA scans/allports <IP>

# 4) Nos serviços encontrados: scripts específicos + UDP provável.
nmap -sV -sC -p<portas_abertas> -oA scans/targeted <IP>
sudo nmap -sU --top-ports 50 -oA scans/udp <IP>

# 5) Versão exata + SO apenas quando necessário (é lento).
sudo nmap -A -p<portas_abertas> -oA scans/aggressive <IP>
```

---

## ⚠️ Pegadinhas

| 🕳️ | Detalhe |
|-----|---------|
| Sem root | `-sS`, `-O`, `-sU` precisam de privilégio; sem root o Nmap cai para `-sT`. |
| Confiar só no top 1000 | Use `-p-` uma vez. Serviços em 8080/8443/5985 escapam do default. |
| Ping bloqueado | Host "morto" que só bloqueia ICMP. Use `-Pn`. |
| Scan UDP completo | `-sU` sem filtro demora horas. Sempre `--top-ports` + `-T4`. |
| `-T5` em rede real | Perde pacotes e gera falso "closed". Em lab tudo bem, em cliente não. |
| `--script vuln` sem aviso | Pode ser intrusivo e disparar alerta de SOC. |
| Esquecer `-oA` | Sem saída salva você repete o scan e perde comparação temporal. |
| Decoys no seu próprio IP | Você vira alvo de ban; mantenha `-T2` junto de `-f`/`--data-length`. |
| Sobrescrever scan | Reusar o mesmo nome de saída apaga o anterior; versiona com data. |

---

## ✅ Checklist rápido

- [ ] Host descoberto e confirmado (`-sn` ou `-Pn`)
- [ ] Passe rápido (`-sV -sC`) salvo
- [ ] Scan completo `-p-` com `--min-rate` salvo
- [ ] UDP relevante testado (`--top-ports`)
- [ ] Cada porta aberta tem versão/banner anotada
- [ ] Scripts NSE específicos ao serviço executados
- [ ] Estado das portas filtradas registrado
- [ ] Saída em `-oA` com nome versionado
- [ ] XML convertido para HTML quando for ao relatório
- [ ] Próximo passo: enumeração detalhada do serviço ([[CPTS 04 - Footprinting]])

---

## 🔗 Relacionado

- [[CPTS 04 - Footprinting]]
- [[CPTS 02 - Getting Started]]
- [[Fase 03 - Reconhecimento de Rede]]
- [[CPTS 11 - Attacking Common Services]]
- [[Fase 04 - Enumeração de Serviços]]
