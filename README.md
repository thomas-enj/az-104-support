# Support de révision AZ-104

## Microsoft Azure Administrator

Guide en français, avec les noms de services, ressources, rôles et menus conservés en anglais pour correspondre au portail et à l'examen.

**Référence :** compétences mesurées par Microsoft à partir du 17 avril 2026. Consultez toujours le [study guide officiel AZ-104](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104) avant l'examen.

## Parcours du guide

| Module | Domaine | Pondération |
|---|---|---:|
| [01 - Identities and governance](modules/01-identities-governance.md) | Manage Azure identities and governance | 20-25 % |
| [02 - Storage](modules/02-storage.md) | Implement and manage storage | 15-20 % |
| [03 - Compute](modules/03-compute.md) | Deploy and manage Azure compute resources | 20-25 % |
| [04 - Virtual networking](modules/04-virtual-networking.md) | Implement and manage virtual networking | 15-20 % |
| [05 - Monitoring and recovery](modules/05-monitoring-recovery.md) | Monitor and maintain Azure resources | 10-15 % |

Le score de réussite est **700 ou plus**. L'examen évalue surtout la capacité à choisir et configurer le bon service dans un scénario.

## Prérequis

- `Azure portal`, `Azure CLI`, `Azure PowerShell`
- Bases des systèmes d'exploitation, réseaux, serveurs et virtualisation
- `Microsoft Entra ID`
- `Azure Resource Manager (ARM)` et fichiers `Bicep`
- Régions, `Availability Zones`, `Availability Sets` et modèle de responsabilité partagée

## Méthode de préparation

1. Lire les cinq modules une première fois.
2. Refaire les labs dans un abonnement de test ou un sandbox Microsoft Learn.
3. Pour chaque service, savoir le créer dans le Portal, le sécuriser et diagnostiquer son erreur courante.
4. Faire le [Practice Assessment AZ-104](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/practice/assessment?assessment-type=practice&assessmentId=21).
5. Revoir les erreurs, puis utiliser l'[exam sandbox](https://aka.ms/examdemo).

### Réflexe devant une question

1. Identifier le scope : `management group`, `subscription`, `resource group` ou `resource`.
2. Distinguer `control plane` et `data plane`.
3. Repérer la contrainte déterminante : coût, latence, région, SLA, sécurité, RPO/RTO ou exploitation.
4. Éliminer les solutions qui donnent trop de privilèges ou ajoutent un composant inutile.
5. Relire le verbe de la question : *allow*, *deny*, *audit*, *restore*, *scale*, *route* ou *resolve*.

## Distinctions à mémoriser

| Ne pas confondre | Différence décisive |
|---|---|
| Microsoft Entra ID / AD DS | Identité cloud / annuaire de domaine traditionnel |
| RBAC / Azure Policy | Autorisation d'un principal / conformité d'une configuration |
| Owner / Contributor | Contributor ne gère pas les role assignments |
| Management plane / Data plane | Ressource / données contenues dans la ressource |
| LRS / ZRS / GRS | Local / zones / région secondaire |
| SAS / access key | Délégation limitée / secret très puissant du compte |
| Blob snapshot / Backup | Version ponctuelle / protection et restauration gouvernées |
| Scale up / Scale out | Plus de puissance / plus d'instances |
| Availability Set / Availability Zone | Fault/update domains / zones physiques séparées |
| ACI / Container Apps / AKS | Container ponctuel / application serverless / Kubernetes |
| NSG / Azure Firewall | Filtrage subnet/NIC / firewall managé centralisé |
| Service endpoint / Private endpoint | Endpoint public restreint / IP privée dédiée |
| Azure Load Balancer / Application Gateway | L4 TCP/UDP / L7 HTTP(S) |
| Metrics / Logs | Série numérique / événements analysés avec KQL |
| Azure Backup / Site Recovery | Restore de données / reprise et failover |

## Checklist finale

- [ ] Je peux choisir le bon module et le bon service à partir d'un scénario.
- [ ] Je sais expliquer le scope, le rôle et le niveau d'accès minimal.
- [ ] Je sais créer les ressources principales dans le Portal anglais.
- [ ] Je sais utiliser Azure CLI, PowerShell ou Bicep pour les tâches courantes.
- [ ] Je sais diagnostiquer avec `Activity log`, `Azure Monitor` et `Network Watcher`.
- [ ] Je connais les différences `RBAC/Policy`, `Backup/Site Recovery`, `Service endpoint/Private endpoint` et `Load Balancer/Application Gateway`.
- [ ] J'ai refait au moins un lab de chaque module.

## Ressources officielles

- [Study guide AZ-104](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/az-104)
- [AZ-104 Practice Assessment](https://learn.microsoft.com/en-us/credentials/certifications/exams/az-104/practice/assessment?assessment-type=practice&assessmentId=21)
- [Exam sandbox](https://aka.ms/examdemo)
- [Parcours Microsoft Learn AZ-104](https://learn.microsoft.com/en-us/training/paths/az-104-manage-identities-governance/)
- [Azure documentation](https://learn.microsoft.com/en-us/azure/)

### Note de fraîcheur

Les noms, options du Portal, limites, régions disponibles et versions d'API évoluent. Pour une limite ou une disponibilité, la documentation du service prime sur toute fiche de mémorisation.

[Retour en haut](#guide-ultime-de-révision-az-104)