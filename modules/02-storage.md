# Module 02 - Implement and manage storage

[Retour au README](../README.md) | [Module suivant : Compute](03-compute.md)

## Storage accounts

Pour la majorité des scénarios, choisir `StorageV2 (general-purpose v2)`. Relier chaque choix au besoin :

- `Performance` : `Standard` ou `Premium`.
- `Redundancy` : `LRS`, `ZRS`, `GRS`, `RA-GRS`, `GZRS` ou `RA-GZRS`.
- `Default access tier` : `Hot`, `Cool`, `Cold` ou `Archive`.
- `Secure transfer required` : préférer HTTPS.
- `Allow Blob anonymous access` : désactiver sauf exigence explicite.
- `Hierarchical namespace` : capacités Data Lake Storage Gen2.
- `Public network access`, `firewall`, `virtual network rules` et `private endpoints`.

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

Endpoint Blob standard : `https://<storage-account>.blob.core.windows.net`. Les endpoints diffèrent pour Files, Queue, Table et Data Lake.

## Accès au stockage

Ordre de préférence général : identité Microsoft Entra ID avec permissions data plane, puis `SAS` à durée et portée minimales. Les `access keys` donnent un accès très large au compte.

- `Shared access signature (SAS)` : ressource, permissions, période et éventuellement IP/protocole.
- `Stored access policy` : policy nommée sur un container/file share pour gérer ou révoquer les SAS associées.
- Blob data plane : `Storage Blob Data Reader`, `Storage Blob Data Contributor`, `Storage Blob Data Owner`.
- Azure Files : comprendre les permissions share-level, NTFS et `Microsoft Entra Kerberos` selon le scénario.

**Piège :** `Storage Account Contributor` gère le compte mais ne donne pas nécessairement la lecture des blobs.

### SAS et clés

- `Service SAS` est adaptée à un service ou container précis ; `User delegation SAS` est signée avec des credentials Microsoft Entra et convient à Blob/Data Lake ; `Account SAS` couvre les services autorisés du compte.
- Dans une SAS, `sp` représente les permissions, `st` le début, `se` l'expiration, `sip` la plage IP, `spr` le protocole et `sv` la version.
- Utiliser HTTPS pour créer et distribuer une SAS, une durée courte et le moindre privilège. Un léger décalage d'horloge peut rendre une SAS « not yet valid » ; omettre le début ou le placer quelques minutes dans le passé.
- Une `Stored access policy` permet de révoquer les permissions d'une service SAS sans régénérer les access keys.
- Pour une opération d'écriture très risquée, un middle-tier qui authentifie, valide et audite peut être préférable à une SAS directe.
- Les deux storage access keys doivent être gérées dans `Azure Key Vault` et régulièrement régénérées, avec rotation automatisée si possible.

## Blob Storage et protection des données

### Blob Storage

Hiérarchie : `Storage account > Container > Blob`. Types : `Block blob`, `Append blob`, `Page blob`.

- `Hot` : accès fréquent ; `Cool` et `Cold` : accès moins fréquent ; `Archive` : hors ligne et réhydratation avant lecture.
- `Soft delete` protège contre suppression accidentelle.
- `Blob versioning` conserve les versions ; `Container soft delete` protège les containers supprimés.
- `Snapshots` fournissent des versions ponctuelles.
- `Lifecycle management` déplace ou supprime selon âge, préfixes et index tags.
- `Object replication` réplique des blobs entre comptes compatibles ; ce n'est pas un remplacement général de Backup.
- Une container `Public access level` peut être `Private`, `Blob` (lecture anonyme des blobs sans listing) ou `Container` (lecture et listing). L'accès anonyme doit aussi être autorisé au niveau du compte via `Allow Blob anonymous access`.
- Pour réhydrater un blob `Archive`, `Copy Blob` vers un nouveau blob online est le choix à reconnaître ; `Set Blob Tier` peut aussi être utilisé selon le scénario.
- `Object replication` requiert `Blob versioning` sur la source et la destination et ne réplique pas les blob snapshots.
- Les transitions de tier et les suppressions anticipées peuvent générer des frais ; les tiers froids réduisent le coût de stockage mais augmentent généralement le coût d'accès. Vérifier les durées minimales actuelles avant de mémoriser un nombre.

### Encryption et sécurité avancée

- `Storage Service Encryption (SSE)` chiffre automatiquement les données au repos avec AES 256-bit, sans coût ni dégradation de performance significative.
- `Secure transfer required` impose des connexions sécurisées ; désactiver les anciennes versions TLS selon la configuration du compte.
- `Platform-managed keys (PMK)` sont gérées par Azure ; `Customer-managed keys (CMK)` sont créées, contrôlées, auditées, désactivées et tournées par le client dans `Azure Key Vault`.
- Pour une CMK, le storage account et le Key Vault doivent être dans la même région, mais peuvent être dans des subscriptions différentes.
- `Infrastructure encryption` ajoute une seconde couche de chiffrement avec un autre algorithme et une autre clé, en complément de SSE.
- `Storage Insights` fournit l'historique de performance, capacité, disponibilité, metrics et logs. `Microsoft Defender for Storage` ajoute la détection proactive des menaces, notamment malware scanning selon les fonctionnalités activées.

## Azure Files et outils

- `Azure Files` fournit des `file shares` via SMB ou NFS selon le compte et la configuration ; l'accès HTTP/REST est également possible. Il fournit de vrais répertoires accessibles depuis plusieurs VM, contrairement à la hiérarchie logique des blobs.
- Une file share peut atteindre jusqu'à `100 TiB` et un fichier individuel jusqu'à `4 TiB` selon le type de share et les limites actuelles du service.
- `Premium file shares` utilisent `FileStorage`, des SSD et une capacité provisionnée ; `Transaction optimized`, `Hot` et `Cool` ciblent d'autres profils.
- L'accès SMB depuis on-premises utilise le port TCP `445`, souvent bloqué par un firewall ou un ISP.
- L'authentification identity-based via on-premises AD DS, `Microsoft Entra Domain Services` ou `Microsoft Entra Kerberos` est préférable aux access keys partagées.
- Une `file share snapshot` est un point-in-time incrémental en lecture seule ; les snapshots sont supprimés avec la share.
- `Soft delete` des file shares se configure au niveau du compte avec une rétention configurable ; il protège uniquement durant cette fenêtre.
- `Azure File Sync` met en cache une Azure file share sur des Windows Servers. Un `sync group` possède un cloud endpoint et des server endpoints ; jusqu'à 50 server endpoints sont supportés par sync group selon les limites actuelles. `Cloud tiering` conserve localement les données récentes. Un server endpoint doit être sur un volume NTFS enregistré et ne peut pas être le system volume.
- `Azure Storage Explorer` est l'interface graphique multi-service.
- Pour connecter Storage Explorer à un compte externe, connaître le storage account name et la clé de compte, souvent `key1` dans le Portal.
- `AzCopy` est l'outil performant pour copier les données.

Pour un transfert très volumineux lorsque la bande passante est insuffisante, utiliser `Azure Data Box Disk` plutôt qu'un upload réseau prolongé.

```bash
az storage account create --name <name> --resource-group <rg> --location <region> --sku Standard_LRS
az storage container create --name <container> --account-name <account> --auth-mode login
az storage blob upload --account-name <account> --container-name <container> --name <blob> --file <file> --auth-mode login
azcopy copy "<source>" "https://<account>.blob.core.windows.net/<container>?<SAS>" --recursive
```

## Lab

1. Créer un `StorageV2` `Standard` avec `Secure transfer required`.
2. Créer un `container` et charger un blob.
3. Tester un rôle Blob data plane et une SAS en lecture seule.
4. Activer `Blob soft delete`, versioning et une règle `Lifecycle management`.
5. Comparer `service endpoint` et `private endpoint` dans [le module réseau](04-virtual-networking.md).

## Ressources

- [Storage account overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)
- [Azure Blob Storage documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/)
- [Azure Files documentation](https://learn.microsoft.com/en-us/azure/storage/files/)