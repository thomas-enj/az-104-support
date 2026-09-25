# Module 05 - Monitor and maintain Azure resources

[Retour au README](../README.md) | [Module précédent : Virtual networking](04-virtual-networking.md)

## Vue d’ensemble
Ce module couvre la surveillance, l’analyse des incidents et la continuité d’activité. Il permet de comprendre comment Azure identifie les problèmes, les signale et les corrige, ainsi que la manière de protéger les données et les workloads contre la perte ou l’interruption.

## Plan du module
1. Azure Monitor, métriques et journaux
2. Alertes, actions et diagnostics
3. Azure Backup et restauration
4. Azure Site Recovery et reprise d’activité

## Azure Monitor

`Azure Monitor` collecte, analyse et exploite la télémétrie Azure et hybride.

- `Metrics` : séries numériques temporelles, utiles pour seuils et autoscale.
- `Logs` : événements et traces analysés dans `Log Analytics workspace` avec `Kusto Query Language (KQL)`.
- `Activity log` : opérations de management au niveau subscription.
- `Resource logs` : logs détaillés envoyés via `Diagnostic settings` vers Log Analytics, Storage ou Event Hubs ; ils ne sont pas collectés par défaut.
- `Alerts` : metric, log search, activity log, `Smart detection` et autres signaux.
- `Action groups` : email, SMS, push, webhook, Logic App ou Automation Runbook. Un point d'examen important : si une méta-règle déclenche une alerte par minute, le nombre d'emails suit le nombre de déclenchements, alors que le nombre de SMS est plafonné par la politique Azure et peut être envoyé moins souvent qu'une alerte toutes les 5 minutes.
- `Alert processing rules` : suppression ou traitement conditionnel.
- `Insights` : vues spécialisées VM, Storage, réseaux et autres ressources.

`Azure Monitor Agent (AMA)` et `Data Collection Rules (DCR)` contrôlent la collecte depuis les VM.

### Choisir la bonne plateforme de données

- `Metrics` sont stockées dans une base temporelle et servent aux seuils, tendances et alertes rapides.
- `Logs` et traces sont stockés dans un `Log Analytics workspace` et interrogés avec `KQL` pour l'analyse détaillée et la recherche de cause racine.
- Un `Azure Monitor workspace` est une ressource différente, destinée notamment aux métriques Prometheus et OpenTelemetry ; il ne remplace pas un Log Analytics workspace pour les requêtes KQL.
- Les métriques et l'`Activity log` ont une rétention par défaut limitée ; pour conserver les journaux plus longtemps et les interroger, configurer une destination adaptée et la rétention du `Log Analytics workspace`. Les durées et limites évoluent selon le type de donnée et le plan tarifaire.

### KQL à reconnaître

```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| summarize Count = count() by OperationNameValue, ActivityStatusValue
| order by Count desc
```

Autres opérateurs à reconnaître : `search "erreur"` recherche un terme dans plusieurs tables, `where` filtre les lignes et `project` limite les colonnes retournées.

Pour les questions d'architecture, le bon réflexe est : `Metrics` pour un seuil rapide, `Logs`/`KQL` pour correler des événements, `Activity log` pour qui a fait quoi, `Action groups` pour l'alerte et la notification, et `Network Watcher` pour le diagnostic réseau.

Ne pas confondre le signal et son action : une `metric alert` ou une `log alert` détecte une condition, tandis qu'un `Action group` notifie ou déclenche une automatisation. Les `Alert processing rules` modifient le traitement d'une alerte déjà déclenchée.

```kusto
Heartbeat
| summarize LastSeen = max(TimeGenerated) by Computer
| where LastSeen < ago(10m)
```

Pour des resource logs Storage envoyés vers Log Analytics, les tables peuvent inclure `StorageBlobLogs` ou `StorageFileLogs` selon les catégories et le mode de collecte activés. Exemple de recherche des opérations Blob récentes :

```kusto
StorageBlobLogs
| where TimeGenerated > ago(24h)
| summarize Operations = count() by OperationName, StatusText
| order by Operations desc
```

Le nom et la disponibilité des tables dépendent du schéma de diagnostic choisi ; vérifier la table proposée dans le workspace avant d'utiliser la requête.

Réflexe : metrics pour un seuil immédiat ; logs/KQL pour corréler et analyser l'historique.

Une `metric alert` évalue généralement des données de plateforme presque en temps réel. Une `log alert` dépend de l'ingestion des logs et de l'exécution planifiée de la requête KQL ; elle peut donc déclencher plus tard.

`Autoscale` s'appuie sur Azure Monitor mais constitue une action de scaling, pas une notification. Une règle peut combiner métrique et horaire ; vérifier séparément les actions de scale-out, de scale-in, le cooldown et les limites du service.

## Azure Backup

- `Azure Backup` protège et restaure des données ; il n'orchestra pas à lui seul un failover régional.
- La sauvegarde des VM Azure est généralement agentless au niveau de la plateforme ; sur Windows, VSS permet une sauvegarde application-consistent lorsque le workload le prend en charge. Pour quelques serveurs physiques Windows nécessitant un backup simple de fichiers/dossiers directement vers Azure, utiliser le `MARS agent (Microsoft Azure Recovery Services agent)` ; `MABS (Microsoft Azure Backup Server)` implique un serveur de backup. `Azure Site Recovery` est destiné à la réplication et au disaster recovery, pas au simple file/folder backup.
- `Recovery Services vault` stocke notamment les recovery points de VM.
- `Backup vault` est utilisé par certains workloads modernes.
- Une `Backup policy` définit schedule, fréquence et rétention.
- Connaître `soft delete`, `immutable vault`, `security settings`, alerts, restore disk, restore VM et restore files.
- Les choix de protection dépendent du workload, du `RPO` et du `RTO`.

`Soft delete` protège une fenêtre de suppression, `immutable vault` empêche la modification ou la suppression de recovery points selon la configuration, et `resource guard` sépare les opérations critiques. Ces protections complètent la policy de sauvegarde ; elles ne remplacent pas le choix du bon `RPO` et du bon `RTO`.
- Un point d'examen fréquent : le `Recovery Services vault` est régional. Les `Azure file shares` utilisent ce vault lorsque les prérequis de région et de support sont satisfaits ; les blobs et Data Lake Storage utilisent désormais un `Backup vault` pour leurs scénarios de vaulted backup.
- La restauration d'une VM à un état d'il y a 8 jours n'est pas un simple `replace existing` si l'objectif est de minimiser le downtime ; la meilleure approche est souvent une `create new restore configuration` pour restaurer une copie temporaire ou une version en parallèle.

`File Recovery` peut monter temporairement les disques sauvegardés afin de récupérer un fichier sans restaurer toute la VM. Pour déplacer une VM protégée vers un autre resource group, arrêter d'abord la protection en conservant les données (`Stop Backup` avec `Retain Backup Data`), effectuer le déplacement, puis reconfigurer la sauvegarde selon les prérequis du vault.

## Azure Site Recovery

`Azure Site Recovery (ASR)` réplique et orchestre le failover/failback pour la continuité d'activité.

**Backup vs Site Recovery :** Backup restaure des points dans le temps ; Site Recovery maintient une capacité de reprise et orchestre un basculement.

Utiliser `Azure Backup` lorsqu'il faut restaurer des données ou une VM à un point donné. Utiliser `Azure Site Recovery` lorsqu'il faut répliquer un workload, tester un failover et orchestrer le retour vers le site primaire. La sauvegarde et la réplication sont complémentaires.

ASR réplique continuellement les VM vers une région ou un site secondaire. Un `Test failover` démarre la réplique dans un réseau isolé afin de valider le plan de reprise sans interrompre la production ; le failover réel et le failback suivent ensuite le runbook préparé.

### À retenir pour l'opération et le diagnostic

- Un `Recovery Services vault` autorisant la `Multi-User Authorization (MAU)` exige avant tout la création d'un `resource guard`.
- Les notifications d'un `Action group` sont réutilisables et exécutées en parallèle ; les emails, SMS, voix et push sont soumis à un rate limiting Azure. Pour une automatisation fiable, préférer selon le scénario une Logic App, une Function, un webhook sécurisé ou un Event Hub.
- L'`Activity log` montre les opérations de management, tandis que les `Metrics` et les `Logs` portent sur la télémétrie et les événements applicatifs ou plateforme.
- Pour le diagnostic réseau, commencer par `Network Watcher` puis `IP flow verify` ou `Connection Monitor` selon si le besoin est la règle NSG, le routage ou la connectivité.

## Lab

1. Envoyer les `Diagnostic settings` d'une VM vers un Log Analytics workspace.
2. Créer une metric alert et l'associer à un `Action group`.
3. Exécuter deux requêtes KQL dans `Logs`.
4. Créer un `Recovery Services vault` et une `Backup policy`.
5. Protéger une VM de test et effectuer une restauration non destructive.
6. Revoir le workflow `Site Recovery` dans un sandbox.

## Ressources

- [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/overview)
- [Azure Monitor data platform](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/data-platform)
- [Log Analytics workspace overview](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/log-analytics-workspace-overview)
- [Action groups](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/action-groups)
- [Azure Monitor alerts overview](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview)
- [Azure Monitor autoscale overview](https://learn.microsoft.com/en-us/azure/azure-monitor/autoscale/autoscale-overview)
- [Azure Backup overview](https://learn.microsoft.com/en-us/azure/backup/backup-overview)
- [Azure VM Backup introduction](https://learn.microsoft.com/en-us/azure/backup/backup-azure-vms-introduction)
- [Azure Backup support matrix](https://learn.microsoft.com/en-us/azure/backup/backup-support-matrix)
- [Restore Azure VMs with Azure Backup](https://learn.microsoft.com/en-us/azure/backup/backup-azure-arm-restore-vms)
- [Azure Site Recovery overview](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview)
- [KQL documentation](https://learn.microsoft.com/en-us/kusto/query/)