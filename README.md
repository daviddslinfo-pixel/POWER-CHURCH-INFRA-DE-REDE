# POWER-CHURCH-INFRA-DE-REDE
PASSO A PASSO DE CONSTRUÇÃO DE TOPOLOGIA DE REDE FREELANCE
CLIENTE: POWER CHURCH - BELO HORIZONTE, MG
INFRAESTRUTURA: DELL T630 (PROXMOX) + CISCO 9200L + MULTI-WAN FAILOVER
AUTOR: DAVID DA SILVA LOPES - FREELANCE
DATA: 27 de Março de 2026
=============================================================================

ESTÁGIO 1: PLANEJAMENTO E ENDEREÇAMENTO (PRÉ-REQUISITOS)
-----------------------------------------------------------------------------
1.1 Definir o escopo da rede.
1.2 Criar um plano de endereçamento IP para evitar conflitos.
    * WAN Principal (ISP1): 192.168.1.0/24 (Gateway OPNsense: 192.168.1.10).
    * WAN Backup (ISP2): 192.168.18.0/24 (IP OPNsense via DHCP).
    * LAN Interna: 192.168.3.0/24 (Gateway OPNsense: 192.168.3.1).
    * Proxmox Host Local IP: 192.168.3.10.
1.3 Definir VLANs:
    * VLAN 10: Link WAN Principal.
    * VLAN 20: Link WAN Backup.

ESTÁGIO 2: CONFIGURAÇÃO FÍSICA E DO SWITCH (CISCO 9200L)
-----------------------------------------------------------------------------
2.1 Acessar o switch Cisco (via Console ou SSH).
2.2 Criar as VLANs 10 e 20 no banco de dados de VLANs.
2.3 Configurar as portas de acesso dos provedores:
    * Porta 1 (Cabo ISP1): Modo Acesso (Access), VLAN 10.
    * Porta 2 (Cabo ISP2): Modo Acesso (Access), VLAN 20.
2.4 Configurar a porta Trunk para o servidor:
    * Porta 22 (Cabo para Dell T630): Modo Trunk.
    * Porta 23 (Cabo para Dell T630): iDRAC.
    * Permitir VLANs 10 e 20 Tagged (Marcadas) nesta porta.
2.5 Conectar os cabos fisicamente conforme o planejado.

ESTÁGIO 3: CONFIGURAÇÃO DO HOST PROXMOX (DELL T630)
-----------------------------------------------------------------------------
3.1 Instalar o Proxmox VE 9.1 no Dell T630.
3.2 Configurar a rede local (LAN) do Proxmox em `/etc/network/interfaces`:
    * Criar bridge `vmbr0` atada à interface física `vtnet0` (LAN).
    * IP Fixo do Proxmox: 192.168.3.2/24. Gateway: 192.168.3.1.
3.3 Configurar a interface WAN Trunk no Proxmox:
    * Criar bridge `vmbr1` atada à interface física `vtnet1` (Trunk).
    * Não atribuir IP a esta interface física, ela passará apenas o tráfego VLAN.




ESTÁGIO 4: CONFIGURAÇÃO DA VM OPNSENSE E INTERFACES
-----------------------------------------------------------------------------
4.1 Criar a VM do OPNsense (VM ID 100) no Proxmox.
4.2 Adicionar as interfaces de rede à VM:
    * `net0`: Conectar à bridge `vmbr0` (LAN).
    * `net1`: Conectar à bridge `vmbr1` (WAN Trunk). Ativar firewall na interface.
4.3 Iniciar o boot do OPNsense. No terminal serial (console virtual Proxmox):
    * Atribuir interfaces: `vtnet0` como LAN, `vtnet1` como WAN (inicialmente).
    * Configurar o IP da LAN: `192.168.3.1/24`.
4.4 Acessar a WebGUI do OPNsense pelo navegador (`https://192.168.3.1`).
4.5 Criar as interfaces VLAN dentro do OPNsense:
    * Interfaces > Devices > Add:
        * Device: `vlan0.10`. Parent: `vtnet1`. Tag: 10. Desc: WAN_Principal_VLAN.
        * Device: `vlan0.20`. Parent: `vtnet1`. Tag: 20. Desc: WAN_Backup_VLAN.
4.6 Reatribuir as interfaces WAN em Interfaces > Assignments:
    * Mudar WAN principal de `vtnet1` para `vlan0.10`.
    * Adicionar nova interface (opt1) e atribuir a `vlan0.20`. Nomear como `WAN_BACKUP`.
4.7 Configurar IPs das WANs:
    * Interfaces > WAN: Static IPv4. IP: 192.168.1.10/24. Gateway: `192.168.1.1`.
    * Interfaces > WAN_BACKUP: DHCP IPv4 (ou Static IP 192.168.18.10/24).
4.8 Aplicar alterações e validar se ambas as WANs receberam IPs diferentes (Interfaces > Overview).

ESTÁGIO 5: CONFIGURAÇÃO DE MULTI-WAN E FAILOVER (OPNSENSE)
-----------------------------------------------------------------------------
5.1 Configurar Monitoramento de IP nos Gateways (System > Gateways > Configuration):
    * WAN_GW: Monitor IP `8.8.8.8` (Google).
    * WAN_BACKUP_GW: Monitor IP `1.1.1.1` (Cloudflare).
    * Em Advanced (ambos), definir Probe Interval como `1` e Threshold como `3` para failover rápido.
5.2 Criar o Grupo de Gateways (System > Gateways > Groups):
    * Name: `GW_FAILOVER`.
    * WAN_GW: Tier 1.
    * WAN_BACKUP_GW: Tier 2.
    * Trigger Level: Packet Loss or High Latency.
5.3 Ativar o Failover na Regra de Firewall (Firewall > Rules > LAN):
    * Editar a regra "Default allow LAN to any" (IPv4).
    * Em Gateway, selecionar o grupo `GW_FAILOVER`.
5.4 Validar o failover: Desconectar cabo da Porta 1 do switch Cisco e monitorar o ping para `8.8.8.8` (deve ter um pulo de ~3 segundos e voltar).

ESTÁGIO 6: CONFIGURAÇÃO DE ACESSO REMOTO SEGURO (TAILSCALE)
-----------------------------------------------------------------------------
6.1 Instalar Plugin Tailscale no OPNsense (System > Firmware > Plugins).
6.2 Ativar o Tailscale no OPNsense (VPN > Tailscale > Settings) e autenticar.
6.3 Ativar Subnet Router no OPNsense para a rede interna:
    * SSH/Console: `tailscale up --advertise-routes=192.168.3.0/24`
    * Aprovar a rota no Tailscale Admin Console.
6.4 Instalar Tailscale no Host Proxmox (Shell do node 'power'):
    * `curl -fsSL https://tailscale.com/install.sh | sh`
    * Autenticar: `tailscale up --accept-routes=false --accept-dns=false` (Para evitar lockouts).
6.5 Configurar Whitelist no Firewall do Proxmox (Se necessário/ativo):
    * Adicionar regras permitindo tráfego da rede Tailscale (`100.64.0.0/10`) para a porta WebGUI (`8006`) em `/etc/pve/firewall/cluster.fw`.
6.6 Criar Regras de Firewall no OPNsense para permitir tráfego na interface virtual Tailscale.

ESTÁGIO 7: FINALIZAÇÃO E BACKUPS
-----------------------------------------------------------------------------
7.1 Confirmar que todos os serviços de produção (VMs) estão online no Proxmox.
7.2 Validar acesso remoto ao Proxmox WebGUI (`https://100.x.y.z:8006`) e OPNsense WebGUI (`https://100.x.y.z`) estando fora da rede do cliente.
7.3 Realizar Backup da Configuração do OPNsense (System > Configuration > Backups).
7.4 Realizar Backup do Switch Cisco (wr).
7.5 Documentar os IPs do Tailscale e logins de emergência (iDRAC).

=============================================================================
Ambiente Freelance Validado e Operacional.
=============================================================================

