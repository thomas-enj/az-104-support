# Module 02 - Implement and manage storage

[Retour au README](../README.md) | [Module précédent : Identities and governance](01-identities-governance.md) | [Module suivant : Compute](03-compute.md)

## Vue d’ensemble
Ce module traite les services de stockage Azure et les décisions de conception associées. L’objectif est de choisir le bon type de compte, la bonne redondance, le bon niveau d’accès et le bon mécanisme de sécurisation selon le besoin métier.

## Plan du module
1. Comptes de stockage, types et redondance
2. Accès aux données et mécanismes de sécurité
3. Blobs, lifecycle et protection des données
4. Azure Files, Data Lake et File Sync

## Storage accounts

Pour la majorité des scénarios, choisir `StorageV2 (general-purpose v2)`. Relier chaque choix au besoin :

- `Performance` : `Standard` ou `Premium`.
- `Redundancy` : `LRS`, `ZRS`, `GRS`, `RA-GRS`, `GZRS` ou `RA-GZRS`.
- `Default access tier` : `Hot`, `Cool`, `Cold` ou `Archive`.
- `Secure transfer required` : préférer HTTPS.
- `Allow Blob anonymous access` : désactiver sauf exigence explicite.
- `Hierarchical namespace` : capacités Data Lake Storage Gen2.
- `Public network access`, `firewall`, `virtual network rules` et `private endpoints`.

### Nom et type du compte

- Le nom d'un storage account doit être unique dans Azure, comporter entre 3 et 24 caractères et contenir uniquement des lettres minuscules et des chiffres.
- `StorageV2 (general-purpose v2)` est le choix recommandé pour un compte qui doit héberger Blob, Files, Queue et Table. Les comptes Premium sont spécialisés : `BlockBlobStorage`, `FileStorage` et les comptes de `Page blobs`.
- Le type de compte ne se convertit pas librement vers un autre type Premium : créer le compte cible et copier les données si le scénario l'exige.

Azure Storage couvre notamment les données structurées, non structurées et les données de VM. Les services à distinguer sont `Blob Storage` (objets), `Azure Files` (partages), `Queue Storage` (messages asynchrones) et `Table Storage` (données NoSQL clé/valeur).

- `Standard` vise le coût par capacité ; `Premium` utilise des SSD et vise une latence constante.
- `StorageV2 (GPv2)` est le choix généraliste. `Premium page blobs` convient notamment aux disques de VM ; `Premium file shares` convient aux partages haute performance.
- Un compte `Standard` ne se convertit pas directement en compte `Premium` : créer le compte cible et copier les données.
- `Hierarchical namespace (HNS)` est requis pour certaines capacités Data Lake Storage Gen2, notamment le support SFTP. `NFSv3` permet de monter un container compatible comme un partage NFS pour des workloads Linux.
- `Default to Microsoft Entra authorization` rend RBAC Entra prioritaire par rapport aux shared keys lorsque le scénario le permet.

| Redundancy | Résumé |
|---|---|
| `LRS` | Copies dans un seul datacenter/région |
| `ZRS` | Réplication synchrone dans plusieurs availability zones |
| `GRS` | Réplication vers une région secondaire, lecture secondaire non activée par défaut |
| `RA-GRS` | GRS avec lecture depuis le secondaire |
| `GZRS` | ZRS primaire + réplication géographique |
| `RA-GZRS` | GZRS avec lecture secondaire |

Ne pas confondre `redundancy` et `backup` : la redondance protège disponibilité/durabilité ; le backup fournit des recovery points.

`LRS` protège surtout contre les pannes matérielles locales. `ZRS` conserve les écritures dans plusieurs zones de la région primaire. `GRS` et `GZRS` répliquent de façon asynchrone vers une région secondaire ; le RPO n'est donc pas nul. `RA-GRS` et `RA-GZRS` ajoutent la lecture depuis le secondaire, mais ne rendent pas automatiquement les écritures disponibles dans cette région. Les options et les régions compatibles dépendent du type de compte.

La redondance répond à un besoin de disponibilité et de durabilité ; elle ne remplace ni `Azure Backup` ni une stratégie de restauration. Si deux workloads ont des exigences de résilience différentes, les placer dans des comptes distincts car la redondance est un réglage du compte.

Avec `GRS` ou `GZRS`, un `account failover` peut promouvoir la région secondaire en région primaire lors d'une panne régionale. Cette opération peut entraîner une perte des écritures encore en attente de réplication ; le compte utilise alors une redondance locale dans la nouvelle région primaire. `RA-GRS` et `RA-GZRS` permettent la lecture secondaire avant un failover, mais pas l'écriture normale dans cette région.

Endpoint Blob standard : `https://<storage-account>.blob.core.windows.net`. Les endpoints diffèrent pour Files, Queue, Table et Data Lake.

### Choisir le bon service Storage

| Service | Modèle | Cas d'usage AZ-104 |
|---|---|---|
| `Blob Storage` | Objets non structurés | Fichiers, médias, sauvegardes, data lake |
| `Azure Files` | Partage de fichiers managé | Partage SMB/NFS monté par plusieurs machines |
| `Queue Storage` | Messages asynchrones | Découpler un producteur et un worker simple |
| `Table Storage` | NoSQL clé/valeur | Données structurées sans schéma relationnel |

`Queue Storage` ne garantit pas le même modèle de messagerie que `Azure Service Bus` : retenir Queue pour une file simple et économique, et Service Bus pour les sessions, transactions, topics/subscriptions ou dead-lettering avancé. Les messages Queue ont un `visibility timeout` ; un message qui échoue peut être déplacé par l'application vers une `poison queue` conventionnelle.

`Table Storage` utilise `PartitionKey` et `RowKey`. La partition est la principale unité de scalabilité et de routage : éviter une seule partition chaude et choisir des clés répartissant les écritures. `Cosmos DB for Table` est une alternative lorsque le scénario exige les capacités globales et les SLA de Cosmos DB.

## Accès au stockage

Ordre de préférence général : identité Microsoft Entra ID avec permissions data plane, puis `SAS` à durée et portée minimales. Les `access keys` donnent un accès très large au compte.

- `Shared access signature (SAS)` : ressource, permissions, période et éventuellement IP/protocole.
- `Stored access policy` : policy nommée sur un container/file share pour gérer ou révoquer les SAS associées.
- Blob data plane : `Storage Blob Data Reader`, `Storage Blob Data Contributor`, `Storage Blob Data Owner`.
- Azure Files : comprendre les permissions share-level, NTFS et `Microsoft Entra Kerberos` selon le scénario.

**Piège :** `Storage Account Contributor` gère le compte mais ne donne pas nécessairement la lecture des blobs.

Pour un accès applicatif, préférer une identité managée avec un rôle data plane au scope minimal. Les `access keys` et les SAS signées par une clé de compte doivent rester des solutions limitées et faire l'objet d'une rotation.

### SAS et clés

- `Service SAS` est adaptée à un service ou container précis ; `User delegation SAS` est signée avec des credentials Microsoft Entra et convient à Blob/Data Lake ; `Account SAS` couvre les services autorisés du compte.
- Dans une SAS, `sp` représente les permissions, `st` le début, `se` l'expiration, `sip` la plage IP, `spr` le protocole et `sv` la version.
- Utiliser HTTPS pour créer et distribuer une SAS, une durée courte et le moindre privilège. Un léger décalage d'horloge peut rendre une SAS « not yet valid » ; omettre le début ou le placer quelques minutes dans le passé.
- Une `Stored access policy` permet de révoquer les permissions d'une service SAS sans régénérer les access keys.
- Une `User delegation SAS` est préférable à une SAS signée par une clé de compte lorsque le client peut s'authentifier avec Microsoft Entra. Pour une SAS de compte ou de service signée avec une access key, planifier la rotation des clés et limiter la durée de validité.
- Pour une opération d'écriture très risquée, un middle-tier qui authentifie, valide et audite peut être préférable à une SAS directe.
- Les deux storage access keys doivent être gérées dans `Azure Key Vault` et régulièrement régénérées, avec rotation automatisée si possible.

### Managed identity

Une `system-assigned managed identity` suit le cycle de vie de sa ressource ; une `user-assigned managed identity` peut être réutilisée par plusieurs ressources. Pour une VM, une Function ou un App Service qui lit des blobs, activer l'identité puis lui attribuer, au scope minimal, `Storage Blob Data Reader` ou `Storage Blob Data Contributor`. L'application obtient ensuite un token Microsoft Entra sans stocker de secret ni d'access key. Cette identité doit avoir un rôle data plane : `Storage Account Contributor` seul ne permet pas de lire les blobs.

## Blob Storage et protection des données

### Blob Storage

Hiérarchie : `Storage account > Container > Blob`. Types : `Block blob`, `Append blob`, `Page blob`.

- `Hot` : accès fréquent ; `Cool` et `Cold` : accès moins fréquent ; `Archive` : hors ligne et réhydratation avant lecture.
- `Smart` : pour un compte GPv2 avec redondance zonale lorsque le profil d'accès est incertain ; Azure déplace automatiquement les block blobs entre `Hot`, `Cool` et `Cold`. `Smart` ne prend pas en charge `Archive`, les comptes Premium, les comptes GPv1 ni les blobs `Append`/`Page`.
- `Soft delete` protège contre suppression accidentelle.
- `Blob versioning` conserve les versions ; `Container soft delete` protège les containers supprimés.
- `Snapshots` fournissent des versions ponctuelles.
- `Lifecycle management` déplace ou supprime selon âge, préfixes et index tags.
- `Object replication` réplique des blobs entre comptes compatibles ; ce n'est pas un remplacement général de Backup.
- Une container `Public access level` peut être `Private`, `Blob` (lecture anonyme des blobs sans listing) ou `Container` (lecture et listing). L'accès anonyme doit aussi être autorisé au niveau du compte via `Allow Blob anonymous access`.
- Pour réhydrater un blob `Archive`, `Copy Blob` vers un nouveau blob online est le choix à reconnaître ; `Set Blob Tier` peut aussi être utilisé selon le scénario.
- `Object replication` requiert `Blob versioning` sur la source et la destination et ne réplique pas les blob snapshots.
- Les transitions de tier et les suppressions anticipées peuvent générer des frais ; les tiers froids réduisent le coût de stockage mais augmentent généralement le coût d'accès. Vérifier les durées minimales actuelles avant de mémoriser un nombre.

| Tier | Accès | Rétention minimale recommandée | Point d'examen |
|---|---|---:|---|
| `Hot` | Fréquent | Aucune | Stockage plus cher, accès moins cher |
| `Cool` | Peu fréquent, accès immédiat | 30 jours | Accès et transactions plus chers |
| `Cold` | Très rare, accès immédiat | 90 jours | Coût de stockage plus bas, accès plus cher |
| `Archive` | Rare, hors ligne | 180 jours | Réhydratation avant lecture, jusqu'à plusieurs heures |

Les tiers d'accès s'appliquent aux `Block blobs`. `Archive` est pris en charge uniquement avec `LRS`, `GRS` ou `RA-GRS` ; il n'est pas compatible avec `ZRS`, `GZRS` ou `RA-GZRS`. Une suppression, réécriture ou sortie anticipée d'un tier froid peut entraîner une pénalité proratisée.

`Archive` est hors ligne : le blob doit être réhydraté avant sa lecture. `Soft delete`, `versioning`, `snapshots`, `object replication` et Backup répondent à des problèmes différents ; aucun de ces mécanismes ne doit être présenté comme un substitut universel aux autres.

La réhydratation standard d'un blob Archive peut prendre plusieurs heures, parfois jusqu'à environ 15 heures ; la priorité élevée réduit ce délai selon la capacité disponible et les conditions du service. Les tiers froids imposent une durée minimale de conservation : une suppression ou une sortie anticipée peut donc générer des frais pour la période restante. Vérifier les durées et les tarifs actuels avant de mémoriser une valeur exacte.

### Lifecycle management

Une règle `Lifecycle management` cible les blobs selon leur préfixe ou leurs index tags. Elle peut déplacer les blobs vers `Cool`, `Cold` ou `Archive`, supprimer les blobs de base, les versions et les snapshots, et s'exécute périodiquement : ce n'est pas une action instantanée. Les conditions peuvent utiliser la date de création, la dernière modification ou la dernière lecture si `last access time tracking` est activé. Une modification de règle peut prendre jusqu'à 24 heures avant sa première exécution.

Exemple de règle à adapter aux limites et coûts actuels du service :

```json
{
	"rules": [
		{
			"enabled": true,
			"name": "archive-logs",
			"type": "Lifecycle",
			"definition": {
				"filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
				"actions": {
					"baseBlob": {
						"tierToCool": { "daysAfterModificationGreaterThan": 30 },
						"tierToArchive": { "daysAfterModificationGreaterThan": 90 },
						"delete": { "daysAfterModificationGreaterThan": 365 }
					}
				}
			}
		}
	]
}
```

Le choix d'un tier ne dépend pas seulement du prix par GiB : tenir compte du coût de lecture, de la latence de réhydratation, des durées minimales et des frais de transition. Une règle agressive peut coûter plus cher si les données sont relues rapidement.

### Immutable blobs (WORM)

L'immuabilité protège des blobs contre la modification et la suppression pendant une durée imposée. Une `time-based retention policy` applique une durée ; un `legal hold` reste actif jusqu'à la suppression explicite des tags légaux. Ces mécanismes répondent à des exigences de conservation et ne remplacent ni la redondance ni un backup. Tester la politique sur un container dédié : pendant la rétention, même un administrateur ne peut pas contourner les restrictions ordinaires.

### Blob backup et restauration

`Azure Backup for Blobs` prend désormais deux formes à distinguer :

- `Operational backup` : protection locale continue fondée sur le point-in-time restore, le soft delete, le change feed et le versioning ; la restauration se fait dans le compte source.
- `Vaulted backup` : copie hors site vers un `Backup vault`, avec planification, rétention longue et restauration vers un autre compte Storage.

Ces mécanismes protègent les block blobs, mais ne remplacent pas la redondance du compte. Ne pas confondre `soft delete` (fenêtre contre une suppression), `versioning` (versions conservées), `snapshot` (copie ponctuelle), `object replication` (copie entre comptes) et Backup (points de restauration gouvernés). Pour un besoin de reprise régionale d'une application et de ses VM, revoir plutôt `Azure Site Recovery` dans le module 05.

### Encryption et sécurité avancée

- `Storage Service Encryption (SSE)` chiffre automatiquement les données au repos avec AES 256-bit, sans coût ni dégradation de performance significative.
- `Secure transfer required` impose HTTPS et le chiffrement des connexions compatibles ; le laisser activé sauf contrainte legacy explicitement justifiée.
- `Platform-managed keys (PMK)` sont gérées par Azure ; `Customer-managed keys (CMK)` sont créées, contrôlées, auditées, désactivées et tournées par le client dans `Azure Key Vault`.
- Pour une CMK, le storage account et le Key Vault doivent être dans la même région, mais peuvent être dans des subscriptions différentes.
- `Infrastructure encryption` ajoute une seconde couche de chiffrement avec un autre algorithme et une autre clé, en complément de SSE.
- `Storage Insights` fournit l'historique de performance, capacité, disponibilité, metrics et logs. `Microsoft Defender for Storage` ajoute la détection proactive des menaces, notamment malware scanning selon les fonctionnalités activées.
- Un `Private Endpoint` fournit une IP privée dans un subnet, mais ne désactive pas à lui seul l'endpoint public. Pour imposer un accès privé, configurer aussi `Public network access = Disabled` ou des règles réseau adaptées, ainsi que la résolution `Private DNS`.

### Redondance, coût et access tiers

- `LRS` conserve trois copies de données dans un seul datacenter de la région et est souvent le mode de base à retenir lorsqu'un objectif de durabilité locale suffit.
- Les tiers `Hot`, `Cool`, `Cold` et `Archive` affectent le coût d'un stockage massif. Pour des données peu sollicités, `Access tier (default)` est la bonne zone de réglage pour réduire les coûts de stockage.
- `Archive` est le niveau le moins cher, mais il impose une réhydratation avant lecture et l'accès est plus lent, ce qui le rend adapté aux données très rares.
- Les règles de `Lifecycle management` doivent tenir compte du coût de lecture, de la latence de réhydratation et des pénalités de transition, pas seulement du prix par GiB.

En examen, ne pas confondre `redundancy`, `backup` et `lifecycle tiering` : la redondance protège la disponibilité/durabilité, le backup fournit des points de restauration, et les access tiers optimisent le coût de données peu utilisées.

Pour un compte Storage qui regroupe plusieurs services, la redondance est un réglage du compte et s'applique à l'ensemble des blobs, files, queues et tables qu'il contient. Si deux workloads ont des exigences de résilience différentes, les séparer dans des comptes distincts.

## Data Lake Storage Gen2

`Azure Data Lake Storage Gen2` est un compte `StorageV2` avec `Hierarchical namespace (HNS)` activé. HNS fournit une hiérarchie de répertoires et des opérations adaptées aux workloads analytiques ; son activation est un choix de conception à faire lors de la création du compte. `SFTP` et, selon la configuration, `NFSv3` sont des capacités à associer à ce modèle.

L'accès combine :

- `Azure RBAC` pour le scope du compte ou du container et les rôles data plane ;
- des `POSIX ACLs` sur les répertoires et fichiers pour un contrôle fin ;
- Microsoft Entra ID ou une identité managée pour éviter les secrets applicatifs.

Piège d'examen : avoir un rôle sur le compte ne suffit pas toujours si une ACL du chemin refuse l'accès. Valider le chemin complet et les permissions de traversée des répertoires. HNS est différent d'un simple container Blob et n'est pas activable après la création du compte.

## Azure Files et outils

- `Azure Files` fournit des `file shares` via SMB ou NFS 4.1. NFS est disponible pour les partages Premium et les clients Linux ; un même partage ne mélange pas SMB et NFS. L'accès HTTP/REST est également possible.
- SMB utilise le port TCP `445`, qui peut être bloqué par un fournisseur d'accès ou un firewall. Pour un accès on-premises, prévoir selon le scénario un VPN Site-to-Site, un VPN Point-to-Site ou ExpressRoute.
- Un partage NFS nécessite un compte `FileStorage` Premium et une architecture réseau adaptée ; l'accès public doit être désactivé lorsqu'il est imposé par le scénario.
- Pour SMB, l'authentification basée sur l'identité peut utiliser AD DS, Microsoft Entra Domain Services ou Microsoft Entra Kerberos ; elle sépare l'identité de l'utilisateur des access keys du compte. Les permissions share-level et NTFS doivent toutes deux autoriser l'accès.
- Une file share peut atteindre jusqu'à `100 TiB` et un fichier individuel jusqu'à `4 TiB` selon le type de share et les limites actuelles du service.
- `Premium file shares` utilisent `FileStorage`, des SSD et une capacité provisionnée ; les partages Standard utilisent un support HDD et les profils `Transaction optimized`, `Hot` ou `Cool` selon le modèle de facturation. Le choix dépend de la latence, des IOPS, du débit et du coût, pas seulement du nom du tier.
- L'accès SMB depuis on-premises utilise le port TCP `445`, souvent bloqué par un firewall ou un ISP.
- L'authentification identity-based via on-premises AD DS, `Microsoft Entra Domain Services` ou `Microsoft Entra Kerberos` est préférable aux access keys partagées.
- Une `file share snapshot` est un point-in-time incrémental en lecture seule ; les snapshots sont supprimés avec la share.
- `Soft delete` des file shares se configure au niveau du compte avec une rétention configurable ; il protège uniquement durant cette fenêtre.
- `Azure File Sync` met en cache une Azure file share sur des Windows Servers. Un `sync group` possède un cloud endpoint et des server endpoints ; `Cloud tiering` conserve localement les données récentes. Un server endpoint doit être sur un volume NTFS enregistré et ne peut pas être le system volume. Plusieurs serveurs peuvent synchroniser le même partage, mais il faut prévoir la gestion des conflits et ne pas le traiter comme un verrouillage distribué général.
- L'agent `Azure File Sync` s'installe sur `Windows Server`, pas sur Linux ni sur un poste client Windows. Pour remplacer un serveur perdu, installer l'agent sur un nouveau serveur, l'enregistrer et recréer le `Server Endpoint` dans le même `Sync Group` ; les données sont resynchronisées depuis le `Cloud Endpoint`.
- Un antivirus qui ignore mal l'attribut `Offline` peut rappeler tous les fichiers hiérarchisés, saturer le cache local et augmenter la consommation réseau. Vérifier sa compatibilité avec `Cloud Tiering`.
- `Azure Storage Explorer` est l'interface graphique multi-service.
- Pour connecter Storage Explorer à un compte externe, connaître le storage account name et la clé de compte, souvent `key1` dans le Portal.
- `AzCopy` est l'outil performant pour copier les données.
- `Azure Data Box`, `Data Box Heavy`, `Data Box Disk` et `Data Box Gateway` répondent à des contraintes différentes de volume, de transport ou de connectivité ; choisir le produit selon le scénario, pas uniquement selon le nom Data Box.

### Diagnostic du stockage

Les `Diagnostic settings` d'un compte Storage envoient les resource logs et metrics vers `Log Analytics`, `Storage` ou `Event Hubs`. Activer uniquement les catégories nécessaires, par exemple les logs Blob ou File en lecture, écriture et suppression, puis analyser les événements dans le module [Monitoring and recovery](05-monitoring-recovery.md). Les diagnostics ne sont pas activés automatiquement par le simple fait d'utiliser `Storage Insights`.

Pour un transfert très volumineux lorsque la bande passante est insuffisante, utiliser `Azure Data Box Disk` plutôt qu'un upload réseau prolongé.

```bash
az storage account create --name <name> --resource-group <rg> --location <region> --sku Standard_LRS
az storage container create --name <container> --account-name <account> --auth-mode login
az storage blob upload --account-name <account> --container-name <container> --name <blob> --file <file> --auth-mode login
azcopy copy "<source>" "https://<account>.blob.core.windows.net/<container>?<SAS>" --recursive
```

### À retenir pour le design et la sécurité

- La `Default routing tier` peut influencer le coût réseau d'un storage account : le routage via le `Microsoft global network` peut être plus coûteux qu'un routage sur Internet selon le scénario.
- L'`encryption type` peut être modifié après création du compte, mais les options plus profondes comme `infrastructure encryption` ou les `customer-managed keys` doivent souvent être décidées au moment de la création.
- Une règle de `Lifecycle management` qui archive les blobs après 90 jours utilise généralement l'action `tierToArchive` et le filtre `prefixMatch` pour cibler un container ou un préfixe précis.
- La politique de restauration en `Blob versioning` n'est pas illimitée ; la rétention configurée détermine la fenêtre de restauration, même si le stockage conserve plusieurs copies sur la région.
- `allowPublicAccess` ne transforme pas automatiquement le compte en design sécurisé ; il faut combiner l'authentification, les ACL, les règles réseau et le plan de protection des données.

## Lab

1. Créer un `StorageV2` `Standard` avec `Secure transfer required`.
2. Créer un `container` et charger un blob.
3. Tester un rôle Blob data plane et une SAS en lecture seule.
4. Activer `Blob soft delete`, versioning et une règle `Lifecycle management`.
5. Créer une `Queue` et tester l'enqueue/dequeue d'un message ; créer une `Table` et insérer une entité avec `PartitionKey` et `RowKey`.
6. Comparer `service endpoint` et `private endpoint` dans [le module réseau](04-virtual-networking.md).
7. Configurer une `Diagnostic setting` vers Log Analytics et vérifier les logs depuis le [module Monitoring](05-monitoring-recovery.md).

## Ressources

- [Storage account overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)
- [Azure Storage redundancy](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Blob access tiers](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-overview)
- [Smart tier for Blob Storage](https://learn.microsoft.com/en-us/azure/storage/blobs/access-tiers-smart)
- [Blob lifecycle management](https://learn.microsoft.com/en-us/azure/storage/blobs/lifecycle-management-overview)
- [Authorize Blob data operations](https://learn.microsoft.com/en-us/azure/storage/blobs/authorize-data-operations-portal)
- [Storage account keys](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-keys-manage?tabs=azure-portal)
- [Storage SAS overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-sas-overview)
- [Azure Blob Storage documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Azure Blobs backup](https://learn.microsoft.com/en-us/azure/backup/blob-backup-overview)
- [Plan an Azure Files deployment](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-planning)
- [Azure Files identity-based authentication](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-active-directory-overview)
- [Azure Files documentation](https://learn.microsoft.com/en-us/azure/storage/files/)
- [Azure Files backup](https://learn.microsoft.com/en-us/azure/backup/azure-file-share-backup-overview)
- [Azure Data Lake Storage vaulted backup](https://learn.microsoft.com/en-us/azure/backup/azure-data-lake-storage-backup-overview)

[Retour au README](../README.md) | [Module précédent : Identities and governance](01-identities-governance.md) | [Module suivant : Compute](03-compute.md)