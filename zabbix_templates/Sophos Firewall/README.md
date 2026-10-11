# Sophos SFOS by SNMP — template para Zabbix

Template de monitoramento para firewalls **Sophos Firewall (SFOS / XG / XGS)** por SNMP v2c, com descoberta de túneis IPsec pelo nome real da conexão, pontos de acesso WiFi, PoE e licenças.

| | |
|---|---|
| **Zabbix** | 7.4 |
| **MIB** | SFOS-FIREWALL-MIB v22 — `1.3.6.1.4.1.2604.5` |
| **SFOS** | 18 ou superior (PoE e temperaturas exigem firmware que exponha a MIB v22) |
| **Protocolo** | SNMP v2c |
| **Template vinculado** | `Network Generic Device by SNMP` (oficial Zabbix) |

---

## Destaques

**O nome da VPN aparece na interface de túnel, sem de-para manual.**
O Sophos nomeia a interface de túnel como `xfrm<N>`, e o SNMP só devolve esse nome de hardware — `ifAlias` vem vazio em todas as interfaces. Acontece que **`N` é o `sfosIPSecVpnConnId`** da conexão IPsec correspondente:

```
sfosIPSecVpnConnName.1  = "FILIAL_A"    ->  xfrm1
sfosIPSecVpnConnName.3  = "FILIAL_B"    ->  xfrm3
sfosIPSecVpnConnName.10 = "MATRIZ_BKP"  ->  xfrm10
```

A descoberta lê a IF-MIB e a tabela de túneis no mesmo walk e cruza as duas pelo número do `xfrm`, nunca pelo índice da linha. O resultado são itens `VPN FILIAL_A: Download` em vez de `Interface xfrm1(): Bits received`, com zero configuração por host.

**Cobertura ampla** — 75 itens e 7 regras de descoberta: CPU, memória, swap, disco, temperaturas, 21 serviços, 9 licenças com dias restantes calculados, HA, ventoinhas, fontes, PoE, túneis IPsec, tráfego por túnel, pontos de acesso WiFi, clientes por SSID.

**Sentinela da própria coleta** — se o SNMP parar, os itens de status de túnel congelam no último valor e o painel continua verde. O item `sophos.ipsec.tunnels.count` existe para quebrar esse silêncio: sem coleta, a trigger de `nodata` dispara.

**Ruído sob controle** — serviços e licenças que o ambiente não usa são desligados por macro de contexto, sem precisar desabilitar item a item.

---

## Requisitos

1. SNMP habilitado no firewall: **Administração > SNMP**, com agente SNMP v2c ativo e uma comunidade de leitura.
2. Origem liberada na ACL do SNMP e na regra de firewall (UDP/161) para o Zabbix server ou proxy.
3. Template oficial `Network Generic Device by SNMP` presente no Zabbix (vem de fábrica no 7.4).

A MIB do fabricante **não** precisa ser instalada no servidor: todos os itens usam OID numérico. Ela só é útil para leitura humana, e está disponível em <https://docs.sophos.com/nsg/sophos-firewall/MIB/SOPHOS-XG-MIB.zip>.

---

## Instalação

1. **Dados coletados > Templates > Importar**, selecione o arquivo `sophos-sfos-by-snmp-zabbix-7.4.json` e marque **Create new** e **Update existing**.
2. Crie o host do firewall com uma **interface SNMP** apontando para o IP de gerência.
3. Vincule o template `Sophos SFOS by SNMP`.
4. Defina a macro `{$SNMP_COMMUNITY}` no host (herdada do template genérico vinculado).

> **Atenção ao atualizar:** não marque *Delete missing* se você tiver feito ajustes direto na interface. Essa opção apaga tudo que existe no template e não está no arquivo.

---

## Macros

### Filtros de descoberta

| Macro | Padrão | Função |
|---|---|---|
| `{$SOPHOS.VPN.NAME.MATCHES}` | `.*` | Túneis IPsec a descobrir |
| `{$SOPHOS.VPN.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Túneis IPsec a ignorar |
| `{$SOPHOS.AP.NAME.MATCHES}` | `.*` | Pontos de acesso a descobrir |
| `{$SOPHOS.AP.NAME.NOT_MATCHES}` | `CHANGE_IF_NEEDED` | Pontos de acesso a ignorar |

O valor `CHANGE_IF_NEEDED` é uma expressão regular que não casa com nada — é a convenção do Zabbix para "não excluir ninguém".

### Limiares

| Macro | Padrão | Função |
|---|---|---|
| `{$SOPHOS.CPU.UTIL.CRIT}` | `90` | Uso de CPU (%) |
| `{$SOPHOS.MEMORY.UTIL.MAX}` | `90` | Uso de memória (%) |
| `{$SOPHOS.SWAP.UTIL.MAX}` | `80` | Uso de swap (%) |
| `{$SOPHOS.DISK.UTIL.MAX}` | `90` | Uso de disco (%) |
| `{$SOPHOS.TEMP.CPU.WARN}` | `75` | Temperatura da CPU, aviso (°C) |
| `{$SOPHOS.TEMP.CPU.CRIT}` | `85` | Temperatura da CPU, crítico (°C) |
| `{$SOPHOS.POE.USAGE.MAX}` | `90` | Consumo da porta PoE, % do limite |
| `{$SOPHOS.LICENSE.EXPIRE.WARN}` | `30` | Dias de antecedência para avisar vencimento |
| `{$SOPHOS.IPSEC.NODATA}` | `10m` | Janela sem coleta da MIB de VPN antes de alertar |

### Liga e desliga por contexto

Em vez de desabilitar itens na mão, use a macro com contexto:

```
{$SOPHOS.SERVICE.MONITOR}              = 1    todos os serviços alarmam
{$SOPHOS.SERVICE.MONITOR:"pop3"}       = 0    menos o POP3
{$SOPHOS.LICENSE.MONITOR:"webserver"}  = 0    licença de Web Server ignorada
```

Já vêm desligados por padrão, por serem os casos mais comuns de ambiente que não usa o recurso: serviços `pop3`, `imap4`, `smtp`, `ftp`, `as` e `ha`, e licenças `mail` e `webserver`.

---

## Coleta

### Itens estáticos (75 no total)

| Grupo | Conteúdo |
|---|---|
| Equipamento | nome, modelo, firmware, versão do Webcat, versão das assinaturas IPS |
| Recursos | CPU (média dos núcleos, via HOST-RESOURCES-MIB), memória usada e total, swap, disco usado e total |
| Temperatura | CPU e NPU — a MIB entrega em décimos de grau, o item converte |
| Serviços | 21 itens, de POP3 a DGD |
| Licenças | 9 módulos × status, data de expiração e **dias para expirar** (calculado) |
| HA | status, estado deste equipamento e do par |
| VPN | conexões IPsec configuradas (sentinela da coleta) |
| WiFi | pontos de acesso gerenciados, pontos de acesso desconectados, clientes conectados |
| Itens mestres | 5 walks sem histórico que alimentam as descobertas |

### Regras de descoberta

| Descoberta | Itens por entidade | Triggers |
|---|---|---|
| **Túneis IPsec** | Ativada, Status da conexão, Tipo de conexão, Túneis ativos | túnel inativo (High), parcialmente ativo (Warning) |
| **Tráfego por túnel IPsec** | Download, Upload — nomeados pela VPN | — |
| **Pontos de acesso WiFi** | Status, Clientes conectados, Modelo, IP, MAC | AP desconectado (Average) |
| **SSID** | Clientes conectados | — |
| **Portas PoE** | Alimentação, Limite, Consumo, Tensão, Corrente | consumo acima do limite (Warning) |
| **Ventoinhas** | Velocidade | — |
| **Fontes de alimentação** | Status | — |

Todos os itens de VPN levam a etiqueta `vpn: <nome da conexão>`, e os de WiFi levam `ap:` e `ssid:` — dá para filtrar por túnel ou por rede em *Dados recentes*, em *Problemas* e no Grafana.

### Triggers estáticas

Disco cheio, memória alta, swap alta, CPU alta, temperatura de CPU em dois níveis, mudança de versão do firmware (informativa), 21 de serviço parado, 9 de licença expirada e 2 da sentinela da coleta IPsec.

---

## Notas de implementação

Pontos em que a MIB ou o Zabbix se comportam de forma contraintuitiva. Valem a leitura antes de alterar o template.

**`Ativada` e `Status da conexão` não são a mesma coisa.** `sfosIPSecVpnActivated` é a chave liga/desliga na configuração; `sfosIPSecVpnConnStatus` é o estado operacional. As duas triggers exigem `Ativada = 1` justamente para que uma conexão desligada de propósito não gere incidente.

**`sfosIPSecVpnActiveTunnel` muda de significado conforme o tipo da conexão.** Em conexão baseada em rota (`tunnel-interface`) ele é sempre 0 e não serve para nada. Em conexão de **acesso remoto** ele é o número de túneis de cliente ativos, ou seja, quantos usuários estão conectados naquele perfil. Nessas conexões o `Status da conexão` fica inativo mesmo com gente conectada, porque não existe túnel permanente — é o `Túneis ativos` que mostra a atividade real.

**`sfosIPSecVpnConnType` devolve 0, valor que a MIB não enumera.** A MIB define apenas `host-to-host(1)`, `site-to-site(2)` e `tunnel-interface(3)`, mas a conexão de acesso remoto responde 0. O mapeamento de valores cobre esse caso como `remote-access`.

**O status do ponto de acesso é invertido.** `APStatusType` é `connected(0)` e `disconnected(1)`, ao contrário da intuição e da maioria das MIBs.

**Não existe item por cliente WiFi, de propósito.** Cliente de WiFi associa e desassocia o tempo todo; uma descoberta em cima disso criaria e destruiria itens sem parar e encheria o banco de órfãos. A tabela de clientes é consumida só como agregado — total e por SSID.

**Nome de campo do `SNMP walk to JSON` não pode ser macro de LLD em protótipo de item.** O Zabbix substitui a macro em todo o parâmetro do passo, inclusive dentro do código JavaScript. Usar `{#WIFI.SSID}` como nome do campo transformaria `linhas[i]['{#WIFI.SSID}']` em `linhas[i]['Rede-Corp']`, a chave deixaria de existir e o item devolveria 0 para sempre — **sem erro nenhum**, parecendo apenas que ninguém usa a rede. Por isso os protótipos usam nome de campo comum (`ssid`, `status`, `name`).

**O walk da tabela de clientes repete o SSID.** Uma linha por cliente significa que dois clientes na mesma rede gerariam chave de item duplicada e a descoberta falharia. O passo JavaScript da regra deduplica antes de devolver.

**`sfosDeviceAPMacAddress` vem em hexadecimal.** O item usa o flag de formato `2` do passo *SNMP walk value*, que converte para o formato de MAC.

**A tabela de pontos de acesso é indexada pelo nome.** O `{#SNMPINDEX}` sai como o nome em decimal pontilhado; o nome legível entra pela macro `{#AP.NAME}`, lida da coluna 1.

**`LAST-UPDATED` da MIB não é confiável.** O arquivo publicado marca dezembro de 2018 mesmo contendo objetos recentes como a tabela de PoE. Na dúvida sobre o que o seu firmware expõe, vale mais um `snmpwalk` na árvore do que o arquivo.

---

## Em equipamento virtual

Appliance virtual não tem ventoinha, fonte, PoE nem sensor de temperatura. Nesses hosts:

- `sophos.health.walk` e `sophos.poe.walk` ficam **Não suportado** — é o comportamento esperado, não é falha;
- as descobertas de ventoinha, fonte e PoE não criam nada;
- os itens de temperatura ficam sem dado.

Para evitar o ruído, desabilite esses itens mestres no host virtual.

---

## Segurança

O campo `sfosIPSecVpnConnDes` (coluna `.3` da tabela de túneis) **não está no template de propósito**. Em firewalls gerenciados pelo Sophos Central observou-se esse campo contendo a mesma cadeia de 64 caracteres aleatórios em todas as conexões, com forte aparência de chave pré-compartilhada. Se for o caso do seu ambiente, qualquer um com a comunidade de leitura consegue lê-la — e gravar isso no histórico do Zabbix só aumentaria a exposição.

Confira comparando o campo *Descrição* da conexão com a *Chave pré-compartilhada* em **VPN > Conexões IPsec**. Independentemente do resultado, restrinja o SNMP por ACL de origem e use uma comunidade exclusiva e somente leitura.

---

## Limitações conhecidas

- **Sem dados de SSL VPN.** A MIB não expõe sessões nem contagem de usuários de SSL VPN. O único objeto relacionado é `sfosSSLVpnService`, que só informa se o serviço está rodando. Para acompanhar sessões, o caminho é syslog. Se o acesso remoto for IPsec, o item `Túneis ativos` da conexão de acesso remoto resolve.
- **SSID sem cliente não é descoberto.** Os SSIDs saem da tabela de clientes, então uma rede configurada e vazia não aparece até alguém se associar. SSIDs já descobertos reportam 0 corretamente.
- **A data de expiração de licença é texto.** O cálculo de dias restantes depende do formato que o firmware devolve; confira os itens `Dias para expirar` depois da primeira coleta.
- **O índice da tabela de túneis é o `ConnId`.** As chaves dos itens usam `{#SNMPINDEX}`. Se o firewall reaproveitar um `ConnId` para outra conexão, o histórico daquele item passa a misturar as duas. Não foi observado em uso normal, mas vale saber.

---

## Licença

Sugestão: MIT. Sophos, SFOS, XG e XGS são marcas da Sophos Ltd.; este projeto não tem vínculo com o fabricante.

---

<sub>Mantido por Pablo Pires de Godoi Silva. Issues e PRs são bem-vindos.</sub>
