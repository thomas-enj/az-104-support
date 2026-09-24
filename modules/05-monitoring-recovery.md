# Module 05 - Monitor and maintain Azure resources

[Retour au README](../README.md) | [Module précédent : Virtual networking](04-virtual-networking.md)

## Azure Monitor

`Azure Monitor` collecte, analyse et exploite la télémétrie Azure et hybride.

- `Metrics` : séries numériques temporelles, utiles pour seuils et autoscale.
- `Logs` : événements et traces analysés dans `Log Analytics workspace` avec `Kusto Query Language (KQL)`.
- `Activity log` : opérations de management au niveau subscription.
- `Resource logs` : logs détaillés envoyés via `Diagnostic settings` vers Log Analytics, Storage ou Event Hubs.
- `Alerts` : metric, log search, activity log, `Smart detection` et autres signaux.
- `Action groups` : email, SMS, push, webhook, Logic App ou Automation Runbook.
- `Alert processing rules` : suppression ou traitement conditionnel.
- `Insights` : vues spécialisées VM, Storage, réseaux et autres ressources.

`Azure Monitor Agent (AMA)` et `Data Collection Rules (DCR)` contrôlent la collecte depuis les VM.

### KQL à reconnaître

```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| summarize Count = count() by OperationNameValue, ActivityStatusValue
| order by Count desc
```

```kusto
Heartbeat
| summarize LastSeen = max(TimeGenerated) by Computer
| where LastSeen < ago(10m)
```

Réflexe : metrics pour un seuil immédiat ; logs/KQL pour corréler et analyser l'historique.

Une `metric alert` évalue généralement des données de plateforme presque en temps réel. Une `log alert` dépend de l'ingestion des logs et de l'exécution planifiée de la requête KQL ; elle peut donc déclencher plus tard.

## Azure Backup

- `Azure Backup` protège et restaure des données ; il n'orchestra pas à lui seul un failover régional.
- Pour quelques serveurs physiques Windows nécessitant un backup simple de fichiers/dossiers directement vers Azure, sans backup server dédié ni besoin application-aware, utiliser le `MARS agent (Microsoft Azure Recovery Services agent)`. `MABS (Microsoft Azure Backup Server)` implique un serveur de backup ; `Azure Site Recovery` est destiné à la réplication et au disaster recovery, pas au simple file/folder backup.
- `Recovery Services vault` stocke notamment les recovery points de VM.
- `Backup vault` est utilisé par certains workloads modernes.
- Une `Backup policy` définit schedule, fréquence et rétention.
- Connaître `soft delete`, `immutable vault`, `security settings`, alerts, restore disk, restore VM et restore files.
- Les choix de protection dépendent du workload, du `RPO` et du `RTO`.

## Azure Site Recovery

`Azure Site Recovery (ASR)` réplique et orchestre le failover/failback pour la continuité d'activité.

**Backup vs Site Recovery :** Backup restaure des points dans le temps ; Site Recovery maintient une capacité de reprise et orchestre un basculement.

## Lab

1. Envoyer les `Diagnostic settings` d'une VM vers un Log Analytics workspace.
2. Créer une metric alert et l'associer à un `Action group`.
3. Exécuter deux requêtes KQL dans `Logs`.
4. Créer un `Recovery Services vault` et une `Backup policy`.
5. Protéger une VM de test et effectuer une restauration non destructive.
6. Revoir le workflow `Site Recovery` dans un sandbox.

## Ressources

- [Azure Monitor overview](https://learn.microsoft.com/en-us/azure/azure-monitor/fundamentals/overview)
- [Azure Backup overview](https://learn.microsoft.com/en-us/azure/backup/backup-overview)
- [Azure Site Recovery overview](https://learn.microsoft.com/en-us/azure/site-recovery/site-recovery-overview)
- [KQL documentation](https://learn.microsoft.com/en-us/kusto/query/)