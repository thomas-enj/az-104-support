# Module 04 - Implement and manage virtual networking

[Retour au README](../README.md) | [Module précédent : Compute](03-compute.md) | [Module suivant : Monitoring](05-monitoring-recovery.md)

## Vue d’ensemble
Ce module couvre le cœur du réseau Azure : adressage, segmentation, sécurité, routage, équilibre de charge et diagnostic. La bonne compréhension du réseau repose sur la connaissance du scope, des chemins de trafic et des mécanismes de filtrage.

## Plan du module
1. VNet, sous-réseaux et adressage IP
2. Sécurité réseau, NSG et ASG
3. DNS, peering et routage
4. Load Balancer, Application Gateway et Bastion
5. Diagnostic réseau avec Network Watcher

## VNet, subnet et adressage

Un `Virtual Network (VNet)` possède des address spaces CIDR et des `subnets` non chevauchants. Choisir un address space qui ne chevauche pas les réseaux on-premises ou les VNets futurs. Une VNet et ses subnets couvrent les availability zones de la région ; on ne crée pas un subnet par zone par défaut.

- Utiliser de préférence des plages privées RFC 1918 : `10.0.0.0/8`, `172.16.0.0/12` ou `192.168.0.0/16`. Deux VNets peuvent avoir des plages qui se chevauchent seulement s'ils ne doivent jamais être reliés ; le peering, le VPN et ExpressRoute exigent des espaces non chevauchants.
- `GatewaySubnet` est réservé au gateway.
- Azure réserve cinq adresses IPv4 dans chaque subnet : la première adresse de la plage, les trois adresses suivantes pour les services Azure et la dernière adresse de la plage. Un subnet `/24` contient donc 251 adresses utilisables ; la dernière adresse n'est pas une adresse de broadcast Azure.
- Un subnet `/29` contient 8 adresses théoriques, dont 3 seulement sont utilisables après les 5 réservations Azure. Dimensionner les subnets avec une marge pour les interfaces réseau, les appliances et les extensions futures.
- `Static` private IP convient aux DNS/domain controllers ; `Static` public IP convient aux endpoints stables.
- Une `Standard public IP` est statique et sécurisée par défaut. Une IP publique dissociée d'une VM n'est pas nécessairement supprimée avec celle-ci ; vérifier les ressources orphelines et leur coût.
- Une NIC utilise une `network interface configuration`, un load balancer une `frontend configuration` et un gateway une `gateway IP configuration`.
- `Basic Load Balancer` et `Basic public IP` ont été retirés le 30 septembre 2025 : utiliser `Standard Load Balancer` et une `Standard public IP` compatible.

Concevoir l'adressage avant de créer les connexions : les espaces qui se chevauchent empêchent généralement le peering, le VPN et ExpressRoute. Réserver aussi les subnets spécialisés comme `GatewaySubnet` et `AzureBastionSubnet` dès la phase de planification.

## NSG, ASG et sécurité

- `Network Security Group (NSG)` filtre l'entrée et la sortie ; la priorité la plus basse est évaluée en premier.
- Un NSG est stateful : la réponse à un flux autorisé est autorisée automatiquement. Une règle doit être ajoutée dans le sens d'initiation du trafic.
- Les règles personnalisées ont une priorité unique entre `100` et `4096`. Les règles par défaut ne se suppriment pas mais peuvent être devancées.
- Les règles par défaut utilisent notamment les priorités `65000` à `65500` : `AllowVNetInBound`, `AllowAzureLoadBalancerInBound`, `DenyAllInbound`, `AllowVNetOutBound`, `AllowInternetOutBound` et `DenyAllOutBound`.
- Sans NSG au subnet ni à la NIC, il n'y a pas de posture de filtrage explicite ; les NSG sont nécessaires pour imposer une règle de sécurité réseau ciblée.
- Un NSG peut être associé au subnet et/ou à la NIC ; vérifier les deux niveaux avec `Effective security rules`.
- Une règle utilise CIDR, `service tag`, `Application Security Group (ASG)`, port et protocole.
- Les ASG expriment les rôles (`Web-ASG -> App-ASG`) sans maintenir des IP individuelles.
- Un ASG et les NIC qui lui sont associées doivent être utilisés dans le même VNet ; le peering ne permet pas d'étendre un ASG à un autre VNet.
- Les `augmented security rules` regroupent plusieurs IP, ports ou plages.
- `Azure Firewall` utilise des `NAT rule collections (DNAT)` pour traduire et filtrer le trafic entrant d'Internet vers des ressources internes. Les `network rules` filtrent le trafic L3/L4 ; les `application rules` filtrent le trafic sortant L7 ; `Threat Intelligence` est un mécanisme distinct.

Exemple : autoriser Internet vers `Web-ASG` sur `80/443`, `Web-ASG` vers `App-ASG` sur `1433`, puis laisser `DenyAllInbound` protéger le reste.

Lorsqu'un NSG est associé au subnet et à la NIC, le trafic entrant doit être autorisé par les deux niveaux ; le trafic sortant est évalué depuis la NIC puis le subnet. Utiliser `Effective security rules` pour voir la règle réellement appliquée. Un NSG autorisé ne suffit pas si le firewall invité de la VM bloque encore le port.

### Service endpoint et Private endpoint

Pour le cas concret d'un compte Storage, le choix et la configuration se lisent d'abord dans [le module Storage](02-storage.md) ; ce module explique le mécanisme réseau, le subnet, le DNS privé et le diagnostic de connectivité.

| | Service endpoint | Private endpoint |
|---|---|---|
| Accès | Endpoint public sécurisé par le VNet | NIC privée dans le subnet |
| Adresse | Le service conserve son endpoint public | Adresse IP privée d'une instance précise |
| DNS | Pas de zone privée généralement nécessaire | `Private DNS zone` souvent nécessaire |
| Usage | Restreindre un service à des subnets | Accès privé à une instance PaaS |

`Azure Bastion` permet RDP/SSH via le Portal sans public IP sur chaque VM. Il nécessite un subnet dédié nommé `AzureBastionSubnet`, généralement dimensionné en `/26` ou plus selon le SKU, et évite d'ouvrir directement RDP/SSH sur Internet. Vérifier le SKU si le scénario exige le partage d'instances, le tunneling ou des fonctionnalités avancées.

Un `service endpoint` conserve l'accès au endpoint public du service et le restreint au VNet ; un `private endpoint` ajoute une NIC et une IP privée pour une instance précise. Dans ce second cas, vérifier la `Private DNS zone` et ne pas supposer que l'accès public est désactivé automatiquement.

## DNS

- `Azure DNS` héberge des zones et records publics.
- Le domaine est enregistré chez un `domain registrar`, puis délégué vers les quatre Azure name servers.
- `A` mappe un nom vers IPv4 ; `AAAA` vers IPv6 ; `CNAME` mappe un nom vers un autre nom et n'est pas utilisable à l'apex ; un `alias record` peut cibler une ressource Azure à l'apex.
- `MX` identifie les serveurs mail ; `TXT` sert notamment à la vérification de domaine, SPF et DKIM ; un enregistrement SPF se crée comme `TXT`.
- `SOA` et `NS` sont créés automatiquement avec une zone.
- Un `CNAME` ne peut pas coexister avec un autre record set du même nom. Le `TTL` est défini au niveau du record set et contrôle la durée de cache.
- Une `Private DNS zone` doit être liée à chaque VNet par un `virtual network link`.
- `Auto-registration` permet à une private DNS zone d'enregistrer automatiquement les VM des VNets liés lorsque l'option est activée.
- La résolution et l'auto-enregistrement sont deux fonctions distinctes : plusieurs VNets peuvent être liés pour résoudre une zone, tandis que l'auto-enregistrement doit respecter les limites actuelles de la zone et une VM ne doit pas être enregistrée automatiquement dans plusieurs zones pour le même nom.
- Si le VNet utilise un `custom DNS server`, les zones privées Azure ne sont pas consultées automatiquement par ce serveur. Prévoir un relais ou une architecture de résolution adaptée ; créer une zone privée ne suffit pas à elle seule.

## VNet peering

- Les deux côtés doivent être configurés : `Initiated` signifie que le second côté manque ; `Connected` indique un peering opérationnel.
- Les VNets ne doivent pas avoir d'address spaces qui se chevauchent.
- `Regional VNet peering` connecte une même région ; `Global VNet peering` connecte des régions différentes via le backbone Azure.
- Le peering n'est pas transitif : A-B et B-C ne donnent pas automatiquement A-C.
- Le trafic entre VNets peerés reste privé sur le backbone Microsoft, sans passerelle ni Internet public. Le peering n'est pas un tunnel VPN : choisir le mécanisme adapté si un chiffrement de tunnel est requis.
- `Gateway transit` permet aux spokes d'utiliser le hub's virtual network gateway ; activer aussi l'option autorisant l'usage du remote gateway selon le scénario.
- `Network Contributor` ou un custom role équivalent est requis ; `Reader` ne suffit pas.
- `Azure Virtual Network Manager` gère les topologies hub-and-spoke ou mesh à grande échelle.
- Un address space peered peut être modifié sans downtime sur la plage existante ; synchroniser ensuite les peers. Les VNets classic ont des contraintes particulières.

## Routage, NVA et BGP

- Azure crée des `system routes` par subnet entre subnets, VNets peered, réseaux connectés et Internet.
- Pour forcer le trafic via une appliance : `route table` + `UDR` comme `0.0.0.0/0 -> Virtual appliance`.
- Un `service tag` dans un UDR (UDR = User-Defined Route) représente un groupe de préfixes IP d'un service Azure, maintenu et mis à jour par Microsoft ; il réduit la maintenance des routes lorsque les adresses changent.
- Azure choisit d'abord le `longest prefix match`. À préfixe égal : `User-defined route`, puis `BGP route`, puis `System route`. Les routes de service endpoint et certaines routes internes ont des règles particulières et ne doivent pas être supposées remplaçables par une UDR.
- Une `NVA` qui transfère du trafic doit avoir `IP forwarding` activé sur sa NIC et être conçue en haute disponibilité.
- `BGP` échange dynamiquement les routes entre on-premises et VPN/ExpressRoute gateway.
- Pour une NVA derrière un `Standard Internal Load Balancer`, `HA ports` (`protocol=All`, `port=0`) répartit tous les flux TCP/UDP.

## Load balancing

- `Azure Load Balancer` est couche 4 TCP/UDP : `frontend IP`, `backend pool`, `health probe`, `load balancing rule`, NAT et outbound rules.
- `Public Load Balancer` expose un frontend public ; `Internal Load Balancer` utilise une adresse privée.
- `Standard Load Balancer` et les `Standard public IP` sont fermés aux connexions entrantes par défaut : les NSG doivent autoriser explicitement le trafic prévu.
- Une health probe retire un backend défaillant des nouvelles connexions ; les connexions existantes peuvent continuer.
- `Session persistence` peut utiliser `Client IP` (2-tuple) ou `Client IP and protocol` (3-tuple).
- `Application Gateway` est couche 7 : `multi-site routing`, `path-based routing`, TLS termination, end-to-end TLS, probes, `connection draining`, autoscaling et `WAF` avec `OWASP Core Rule Sets`.
- Les SKUs Application Gateway V1 ont été retirés depuis le 28 avril 2026 ; retenir `Standard_v2` ou `WAF_v2`.
- Application Gateway doit utiliser un subnet dédié qui ne contient pas d'autres types de ressources. `WAF_v2` ajoute le filtrage applicatif basé sur les règles OWASP ; `Standard_v2` ne fournit pas cette fonction WAF.
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
- Les `NSG flow logs` historiques et `Traffic Analytics` fournissent des tendances et des flux dans le temps, mais les nouveaux NSG flow logs ne sont plus créables et le service est retiré le 30 septembre 2027. Utiliser désormais `Virtual network flow logs`, activés au niveau du VNet, qui peuvent aussi couvrir les règles de sécurité administratives et l'état de chiffrement.
- Pour `Traffic Analytics`, il faut des flow logs, un `storage account` Standard pour les stocker et un `Log Analytics workspace` pour l'analyse et la visualisation. Le compte Storage doit être dans la même région que le VNet/NSG et le workspace peut dépendre des régions supportées par Traffic Analytics.
- Les `Virtual network flow logs` couvrent le VNet mais ne sont pas pris en charge par ACI, Container Apps, App Service et plusieurs services PaaS. Ils ne suppriment donc pas le besoin de diagnostics propres à chaque service.

### À retenir pour la conception réseau

- Un `Azure Firewall Premium` exige une IP publique `Standard` IPv4, `Regional` et `Static` ; les IP `Basic`, IPv6 et `Global` ne sont pas les bonnes réponses.
- Une `Private DNS zone` ne remplace pas forcément un `custom DNS server` si le VNet est explicitement configuré pour l'utiliser. La résolution dépend de la configuration DNS du VNet, pas seulement de l'existence d'une zone privée liée.
- Le `peering` n'est pas transitif : `A-B` et `B-C` n'impliquent pas `A-C` sans peering direct.
- `Azure Bastion` évite d'exposer RDP/SSH directement sur les VM ; il offre une connectivité sécurisée depuis le portail.
- `Service endpoint` et `Private endpoint` ne sont pas équivalents : le premier sécurise l'accès d'un service Azure depuis un VNet, le second fournit une IP privée dédiée à l'instance cible.

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
- [Azure Private DNS](https://learn.microsoft.com/en-us/azure/dns/private-dns-overview)
- [Private DNS virtual network links](https://learn.microsoft.com/en-us/azure/dns/private-dns-virtual-network-links)
- [Virtual network peering](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-peering-overview)
- [Virtual network traffic routing](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview)
- [Azure Load Balancer](https://learn.microsoft.com/en-us/azure/load-balancer/load-balancer-overview)
- [Application Gateway](https://learn.microsoft.com/en-us/azure/application-gateway/overview)
- [Network Watcher](https://learn.microsoft.com/en-us/azure/network-watcher/network-watcher-overview)
- [Azure Bastion overview](https://learn.microsoft.com/en-us/azure/bastion/bastion-overview)
- [Virtual network service endpoints](https://learn.microsoft.com/en-us/azure/virtual-network/virtual-network-service-endpoints-overview)
- [Traffic Analytics](https://learn.microsoft.com/en-us/azure/network-watcher/traffic-analytics)
- [NSG flow logs and retirement](https://learn.microsoft.com/en-us/azure/network-watcher/nsg-flow-logs-overview)
- [Virtual network flow logs](https://learn.microsoft.com/en-us/azure/network-watcher/vnet-flow-logs-overview)

[Retour au README](../README.md) | [Module précédent : Compute](03-compute.md) | [Module suivant : Monitoring](05-monitoring-recovery.md)