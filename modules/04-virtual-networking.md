# Module 04 - Implement and manage virtual networking

[Retour au README](../README.md) | [Module précédent : Compute](03-compute.md) | [Module suivant : Monitoring](05-monitoring-recovery.md)

## VNet, subnet et adressage

Un `Virtual Network (VNet)` possède des address spaces CIDR et des `subnets` non chevauchants. Choisir un address space qui ne chevauche pas les réseaux on-premises ou les VNets futurs. Une VNet et ses subnets couvrent les availability zones de la région ; on ne crée pas un subnet par zone par défaut.

- `GatewaySubnet` est réservé au gateway.
- Azure réserve cinq adresses IPv4 dans chaque subnet : réseau, gateway, deux DNS Azure et dernière adresse.
- `Static` private IP convient aux DNS/domain controllers ; `Static` public IP convient aux endpoints stables.
- Une NIC utilise une `network interface configuration`, un load balancer une `frontend configuration` et un gateway une `gateway IP configuration`.
- `Basic Load Balancer` a été retiré le 30 septembre 2025 : utiliser `Standard Load Balancer` et une `Standard public IP` compatible.

## NSG, ASG et sécurité

- `Network Security Group (NSG)` filtre l'entrée et la sortie ; la priorité la plus basse est évaluée en premier.
- Les règles personnalisées ont une priorité unique entre `100` et `4096`. Les règles par défaut ne se suppriment pas mais peuvent être devancées.
- Sans NSG au subnet ni à la NIC, il n'y a pas de posture de filtrage explicite ; les NSG sont nécessaires pour imposer une règle de sécurité réseau ciblée.
- Un NSG peut être associé au subnet et/ou à la NIC ; vérifier les deux niveaux avec `Effective security rules`.
- Une règle utilise CIDR, `service tag`, `Application Security Group (ASG)`, port et protocole.
- Les ASG expriment les rôles (`Web-ASG -> App-ASG`) sans maintenir des IP individuelles.
- Les `augmented security rules` regroupent plusieurs IP, ports ou plages.
- `Azure Firewall` utilise des `NAT rule collections (DNAT)` pour traduire et filtrer le trafic entrant d'Internet vers des ressources internes. Les `network rules` filtrent le trafic L3/L4 ; les `application rules` filtrent le trafic sortant L7 ; `Threat Intelligence` est un mécanisme distinct.

Exemple : autoriser Internet vers `Web-ASG` sur `80/443`, `Web-ASG` vers `App-ASG` sur `1433`, puis laisser `DenyAllInbound` protéger le reste.

### Service endpoint et Private endpoint

| | Service endpoint | Private endpoint |
|---|---|---|
| Accès | Endpoint public sécurisé par le VNet | NIC privée dans le subnet |
| Adresse | Le service conserve son endpoint public | Adresse IP privée d'une instance précise |
| DNS | Pas de zone privée généralement nécessaire | `Private DNS zone` souvent nécessaire |
| Usage | Restreindre un service à des subnets | Accès privé à une instance PaaS |

`Azure Bastion` permet RDP/SSH via le Portal sans public IP sur chaque VM.

## DNS

- `Azure DNS` héberge des zones et records publics.
- Le domaine est enregistré chez un `domain registrar`, puis délégué vers les quatre Azure name servers.
- `A` mappe un nom vers IPv4 ; `CNAME` mappe un nom vers un autre nom et n'est pas utilisable à l'apex ; un `alias record` peut cibler une ressource Azure à l'apex.
- `SOA` et `NS` sont créés automatiquement avec une zone.
- Une `Private DNS zone` doit être liée à chaque VNet par un `virtual network link`.

## VNet peering

- Les deux côtés doivent être configurés : `Initiated` signifie que le second côté manque ; `Connected` indique un peering opérationnel.
- Les VNets ne doivent pas avoir d'address spaces qui se chevauchent.
- `Regional VNet peering` connecte une même région ; `Global VNet peering` connecte des régions différentes via le backbone Azure.
- Le peering n'est pas transitif : A-B et B-C ne donnent pas automatiquement A-C.
- `Gateway transit` permet aux spokes d'utiliser le hub's virtual network gateway ; activer aussi l'option autorisant l'usage du remote gateway selon le scénario.
- `Network Contributor` ou un custom role équivalent est requis ; `Reader` ne suffit pas.
- `Azure Virtual Network Manager` gère les topologies hub-and-spoke ou mesh à grande échelle.
- Un address space peered peut être modifié sans downtime sur la plage existante ; synchroniser ensuite les peers. Les VNets classic ont des contraintes particulières.

## Routage, NVA et BGP

- Azure crée des `system routes` par subnet entre subnets, VNets peered, réseaux connectés et Internet.
- Pour forcer le trafic via une appliance : `route table` + `UDR` comme `0.0.0.0/0 -> Virtual appliance`.
- Un `service tag` dans un UDR (UDR = User-Defined Route) représente un groupe de préfixes IP d'un service Azure, maintenu et mis à jour par Microsoft ; il réduit la maintenance des routes lorsque les adresses changent.
- Azure choisit le `longest prefix match`. À préfixe égal : `User-defined route`, puis `BGP route`, puis `System route`.
- Une `NVA` qui transfère du trafic doit avoir `IP forwarding` activé sur sa NIC et être conçue en haute disponibilité.
- `BGP` échange dynamiquement les routes entre on-premises et VPN/ExpressRoute gateway.
- Pour une NVA derrière un `Standard Internal Load Balancer`, `HA ports` (`protocol=All`, `port=0`) répartit tous les flux TCP/UDP.

## Load balancing

- `Azure Load Balancer` est couche 4 TCP/UDP : `frontend IP`, `backend pool`, `health probe`, `load balancing rule`, NAT et outbound rules.
- `Public Load Balancer` expose un frontend public ; `Internal Load Balancer` utilise une adresse privée.
- Une health probe retire un backend défaillant des nouvelles connexions ; les connexions existantes peuvent continuer.
- `Session persistence` peut utiliser `Client IP` (2-tuple) ou `Client IP and protocol` (3-tuple).
- `Application Gateway` est couche 7 : `multi-site routing`, `path-based routing`, TLS termination, end-to-end TLS, probes, `connection draining`, autoscaling et `WAF` avec `OWASP Core Rule Sets`.
- Les SKUs Application Gateway V1 ont été retirés le 28 avril 2026 ; retenir `Standard_v2` ou `WAF_v2`.
- Choisir Application Gateway pour WAF/routage HTTP ; Load Balancer pour L4 TCP/UDP ; connaître `Azure Front Door` pour le global L7.

## Network Watcher

`Network Watcher` cible principalement les ressources réseau IaaS, pas le diagnostic applicatif complet d'App Service.

- `Topology` : vue des ressources et relations.
- `Connection Monitor` : connectivité et latence dans le temps.
- `IP flow verify` : `Allowed`/`Denied` et règle NSG responsable.
- `Next hop` : route sélectionnée ; `None` indique généralement un trafic abandonné/non routable.
- `Effective security rules` : agrégation subnet + NIC.
- `Connection troubleshoot` : test ponctuel, avec erreurs comme `NetworkSecurityRule`, `GuestFirewall` ou `DNSResolution`.
- `Packet capture` : capture distante sur VM/VMSS avec filtres.
- `NSG flow logs` et `Traffic Analytics` : tendances et flux dans le temps. Les NSG flow logs sont en retrait au 30 septembre 2027 ; étudier `Virtual network flow logs`.

## Lab

1. Créer un VNet avec `web-subnet` et `management-subnet`.
2. Déployer deux VM et associer un NSG avec règles SSH/RDP limitées et HTTP.
3. Tester `Effective security rules`, `IP flow verify` et `Next hop`.
4. Créer un peering puis un `Private Endpoint` Storage avec `Private DNS zone`.
5. Comparer avec un `service endpoint`.
6. Tester `Connection troubleshoot`, `Connection Monitor` et `Topology`.

## Ressources

- [Virtual Network overview](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-overview)
- [Network security groups](https://learn.microsoft.com/en-us/azure/virtual-network/network-security-groups-overview)
- [Azure DNS overview](https://learn.microsoft.com/en-us/azure/dns/dns-overview)
- [Virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
- [Virtual network traffic routing](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)
- [Azure Load Balancer](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)
- [Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/overview)
- [Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview)