# Module 03 - Deploy and manage Azure compute resources

[Retour au README](../README.md) | [Module suivant : Virtual networking](04-virtual-networking.md)

## Vue d’ensemble
Ce module traite le choix et la mise en œuvre des ressources de calcul Azure. Le point clé est de faire correspondre le service à la charge de travail, puis de dimensionner, sécuriser et surveiller la ressource correctement.

## Plan du module
1. Choix du service de calcul et déploiement déclaratif
2. Machines virtuelles, disques, disponibilité et Spot
3. VMSS, autoscale et haute disponibilité
4. App Service, déploiement et sécurité
5. Conteneurs, ACR, ACI, Container Apps et AKS

## Choisir le service de calcul

Commencer par le niveau de contrôle et le mode d'exécution demandés par le scénario :

| Besoin | Service à retenir | Indice de décision |
|---|---|---|
| Contrôler l'OS, les disques et le réseau | `Azure Virtual Machines` | Administration IaaS et logiciel installé par l'équipe |
| Exécuter plusieurs VM identiques | `Virtual Machine Scale Sets (VMSS)` | Haute disponibilité et autoscale d'un même modèle |
| Héberger une application web ou une API | `Azure App Service` | PaaS, déploiement, slots et TLS sans gérer l'OS |
| Lancer un container ponctuel | `Azure Container Instances (ACI)` | Job ou workload simple sans cluster |
| Exécuter une application containerisée avec ingress et scaling | `Azure Container Apps` | PaaS serverless, revisions et `scale to zero` |
| Administrer Kubernetes et ses API | `Azure Kubernetes Service (AKS)` | Contrôle de l'orchestrateur et de ses extensions |

Réflexe AZ-104 : `scale up` augmente la capacité d'une instance ou change son tier ; `scale out` ajoute des instances. Le choix du service précède le choix du SKU : vérifier la région, les quotas, le SLA, les contraintes réseau, la capacité disque et le coût total.

## ARM et Bicep

`Azure Resource Manager` fournit la couche de déploiement. Un déploiement déclaratif est reproductible et peut être vérifié avant exécution.

Concepts à reconnaître : `resource`, `property`, `resourceId`, `dependsOn`, `parameter`, `variable`, `output`, `scope`, `module` et `what-if`.

```bicep
param location string = resourceGroup().location
param storageName string

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

output storageResourceId string = storage.id
```

```bash
az deployment group what-if --resource-group <rg> --template-file main.bicep --parameters storageName=<name>
az deployment group create --resource-group <rg> --template-file main.bicep --parameters storageName=<name>
az bicep build --file main.bicep
az bicep decompile --file template.json
```

Savoir lire/modifier un ARM template ou Bicep, déployer au bon scope, utiliser `what-if`, exporter un template et convertir ARM vers Bicep. Les références à une autre resource créent souvent une dépendance implicite ; utiliser `dependsOn` pour une dépendance explicite. Choisir le deployment scope (`resourceGroup`, `subscription` ou `managementGroup`) avec soin.

Le `what-if` permet de contrôler les créations, modifications et suppressions avant l'exécution. Le scope du déploiement doit correspondre au scope de la ressource : un template de resource group ne suffit pas pour une assignment au niveau subscription ou management group.

## Virtual machines

Lors de la création, relier `Image`, `VM size`, `Disks`, `NIC`, `VNet/Subnet`, `Public IP`, `NSG`, authentification et `Availability options`.

### Types et familles de VM

Une famille décrit le profil matériel ; une taille précise ajoute le nombre de vCPU, la mémoire, les disques et le débit maximum. Choisir la famille selon la ressource limitante, puis vérifier la taille disponible dans la région et le quota de la subscription.

| Famille | Profil | Exemples de charges |
|---|---|---|
| `B-series` | Burstable, CPU de base avec crédits de burst | Serveur web léger, environnement de développement, petite appliance |
| `D-series` | Usage général, équilibre CPU/mémoire | Serveur applicatif, contrôleur de domaine, application métier |
| `E-series` | Mémoire optimisée | Base de données en mémoire, cache, analyse avec forte consommation RAM |
| `F-series` | Calcul optimisé, davantage de CPU par GiB de mémoire | Traitement batch, serveur web CPU-bound, calcul applicatif |
| `L-series` | Stockage optimisé, débit et IOPS élevés avec disques locaux | NoSQL, data warehouse, traitement de logs ou de données temporaires |
| `N-series` | GPU | IA, rendu graphique, calcul parallèle, visualisation distante |
| `H-series` | HPC, réseau et calcul spécialisés | Simulation, modélisation et workloads scientifiques |
| `M-series` | Très grande capacité mémoire | Bases de données et workloads SAP de grande taille |

Les lettres et suffixes d'une taille ne suffisent pas à déduire toutes ses capacités. Vérifier dans le sélecteur de taille ou la documentation : vCPU, mémoire, nombre et type de data disks, IOPS, MB/s, débit réseau, support des `Availability Zones`, `Premium Storage`, `Accelerated Networking`, `Ephemeral OS disk`, GPU et coût. Une `B-series` peut perdre ses crédits sous charge prolongée ; un disque local temporaire ne remplace pas un managed disk persistant.

**Méthode d'examen :**

1. Identifier la charge dominante : CPU, mémoire, IOPS/débit, GPU ou coût.
2. Éliminer les familles qui ne supportent pas la fonctionnalité exigée dans la région.
3. Vérifier les limites de la taille : nombre de disques, débit, réseau, zones et taille maximale des disques.
4. Contrôler le quota de vCPU et la capacité réelle de la région avant le déploiement.

- `Managed disks` : `Premium SSD`, `Standard SSD`, `Standard HDD`, `Ultra Disk`.
- Distinguer `OS disk` et `data disks`.
- Le `temporary disk` est local à l'hôte et éphémère. Il peut être perdu lors d'un redémarrage, d'une désallocation ou d'une migration de l'hôte ; ne jamais y placer de données critiques. Le disque d'OS et les data disks managés sont les emplacements persistants à privilégier.
- Comparer une taille de VM avec le besoin dominant : vCPU, mémoire, IOPS, débit disque ou réseau. Une taille avec davantage de vCPU ne corrige pas automatiquement un goulot d'étranglement disque.
- Un `snapshot` est une copie ponctuelle ; une `disk image` sert à créer des VM similaires.
- `Availability Zones` séparent les instances dans des zones physiques distinctes.
- `Virtual Machine Scale Sets (VMSS)` gèrent des VM identiques avec autoscaling.
- `Encryption at host` chiffre données temporaires et caches du host, en complément des managed disks.
- Une VM peut être déplacée entre resource groups/subscriptions sous conditions ; changer de VNet implique généralement de la recréer en conservant les disques.

### Automatiser la configuration d'une VM

- `Custom Data` ou `cloud-init` transmet un script lors du provisionnement initial ; il est destiné à l'initialisation et ne remplace pas une exécution récurrente.
- `Custom Script Extension` installe ou exécute un script sur une VM existante.
- `Run Command` exécute ponctuellement PowerShell ou Bash via l'agent Azure, sans ouvrir RDP ou SSH. Pour corriger une VM déjà déployée, préférer `Run Command` ou une extension plutôt que `Custom Data`.

### Azure Spot Virtual Machines

Une `Azure Spot Virtual Machine` utilise la capacité Azure inutilisée avec une remise variable. Elle peut être évincée lorsque Azure récupère la capacité ou lorsque le prix dépasse le `max price`. Elle convient aux traitements batch, au dev/test et aux workloads interruptibles, mais elle n'offre pas de SLA ni de garantie de haute disponibilité.

- `Eviction policy = Deallocate` conserve la VM arrêtée et les disques continuent d'être facturés ; `Delete` supprime la VM et ses disques.
- Une notification d'éviction peut être envoyée environ 30 secondes avant l'éviction, sans garantie de livraison.
- La série `B` n'est pas prise en charge pour Spot ; vérifier la capacité, le quota Spot, la région et le prix courant.
- Une Spot VM ne se convertit pas directement en VM standard : choisir le mode au déploiement.

### VMSS et autoscale

- `Scale out` ajoute des instances ; `Scale in` en retire. Prévoir les deux règles pour absorber la charge puis réduire les coûts.
- Définir `minimum`, `maximum` et `default instance count` avec une marge réelle ; si minimum et maximum sont identiques, aucun scaling dynamique n'est possible.
- Avec une capacité par instance stable, la capacité totale approximative est `instances x capacité d'une instance`. Dimensionner le minimum pour la charge normale, le maximum pour le pic et conserver une marge pour les pannes ou les quotas.
- `Scale-in policy` détermine quelles instances sont retirées. `Max spreading` répartit les instances sur autant de fault domains que possible lorsque la résilience est prioritaire.
- Les VMSS supportent l'autoscale métrique et planifié. Les `Availability Zones` et les `region pairs` répondent à des niveaux de panne différents.
- Les `upgrade policies` contrôlent la propagation d'une nouvelle image ou configuration : `Manual` laisse l'administrateur choisir les instances, `Automatic` applique la mise à jour automatiquement et `Rolling` procède par lots pour conserver une partie de la flotte disponible. Pour une application critique, vérifier aussi les health probes et la taille des lots.
### Disponibilité et reprise des VM

- Un `Availability Set` répartit les VM entre `fault domains` (alimentation et réseau physiques distincts) et `update domains` (groupes redémarrés lors de la maintenance) dans un même datacenter. Il faut au moins deux VM pour bénéficier du SLA associé ; il ne protège pas contre la perte de toute la région.
- Une `Availability Zone` est une zone physique séparée dans une région. Une VM zonale est attachée à une zone ; répartir plusieurs VM sur plusieurs zones protège contre la perte d'un datacenter, si le service et la région le permettent.
- Une `Availability Set` ou une `Availability Zone` doit être choisie lors de la création de la VM ; une VM existante doit généralement être recréée pour changer ce choix. Les valeurs souvent associées sont environ `99,95 %` pour un Availability Set correctement configuré et `99,99 %` pour des VM réparties dans des Availability Zones, mais le SLA dépend de la configuration complète et doit être vérifié dans la documentation actuelle.
- `Azure Backup` restaure des points de récupération ; `Azure Site Recovery` réplique et orchestre un failover/failback vers une autre région ou un autre site.

Diagnostic Portal : `Boot diagnostics`, `Run command`, `Activity log`, `Metrics`, `Disks` et `Networking > Effective security rules`.

## Containers

| Service | Choix adapté |
|---|---|
| `Azure Container Instances (ACI)` | Containers ponctuels, sans orchestration complexe |
| `Azure Container Apps` | Applications containerisées, ingress, revisions et scaling serverless |
| `Azure Container Registry (ACR)` | Registry privé d'images et artefacts |
| `Azure Kubernetes Service (AKS)` | Orchestration Kubernetes |

Dans ACR, connaître `repositories`, `tags`, `admin user` et l'authentification par Entra/managed identity. Le choix du SKU détermine notamment les capacités de réplication, réseau privé et débit ; désactiver l'admin user lorsque l'accès par managed identity suffit.

### Azure Container Instances

- `ACI` convient à un job court, un démarrage rapide et une exposition directe par IP/FQDN sans gérer de cluster.
- Un `container group` avec `Public` IP et port `80` est accessible depuis Internet ; la présence d'une IP publique ne signifie pas une restriction d'accès au système d'exploitation. Le paramètre `restartPolicy` détermine si le conteneur redémarre automatiquement après un échec.
- Pour un `restartPolicy = OnFailure`, le conteneur redémarre automatiquement lorsqu'il tombe en panne. C'est une distinction importante par rapport à un déploiement manuel ou à un `restartPolicy` plus strict.
- Les images ACI doivent être compatibles `x64/AMD64` ; les images `ARM64` ne sont pas prises en charge dans le scénario courant.

### Container groups

- Un `container group` partage le cycle de vie, le réseau local et le port namespace. Deux containers du même groupe ne doivent pas écouter le même port ; changer le port d'écoute du sidecar.
- Le pattern multi-container convient à un sidecar de monitoring qui observe le container principal.

### Ressources, stockage et redémarrage

- Réserver les ressources par container avec un nombre de vCPU et une quantité de mémoire ; la facturation dépend des ressources et du temps d'exécution.
- Les volumes comprennent `emptyDir`, `secret`, `gitRepo` et `Azure Files`. Azure Files fournit la persistance ; le montage de partage Azure Files concerne les containers Linux et utilise les identifiants du compte Storage dans le scénario courant.
- `restartPolicy` peut être `Always`, `Never` ou `OnFailure`. ACI est stateless par défaut : sans volume externe, l'état est perdu lors d'un redémarrage ou d'une suppression.
- `Always` est adapté à un service long-lived ; `OnFailure` relance un conteneur qui se termine en erreur ; `Never` convient à un job ponctuel qui doit s'arrêter après son exécution. Pour un batch, choisir explicitement la policy afin d'éviter une boucle de redémarrage et des coûts inutiles.
- `Never` signifie qu'un conteneur terminé correctement avec le code `0` n'est pas relancé ; un code de sortie non nul peut encore entraîner un redémarrage de la plateforme. Pour garantir une exécution unique, l'application doit gérer l'erreur et terminer selon le comportement attendu.
- Une `managed identity` peut permettre au groupe d'accéder à des services Azure et à ACR sans placer de credentials dans la définition du container.

### Réseau ACI

- L'IP publique et le FQDN d'un container group peuvent changer après un redémarrage ou un nouveau déploiement ; ne pas les traiter comme une IP statique.
- Pour un déploiement dans un VNet, utiliser un subnet dédié. Un `NAT Gateway` est requis pour la connectivité sortante Internet du container group dans cette configuration.
- Les VM offrent une boundary d'isolation plus forte que les containers pour des workloads de tenants strictement séparés.

### Azure Container Apps et AKS

`Azure Container Apps` est une plateforme serverless construite sur Kubernetes qui abstrait l'infrastructure, fournit ingress, revisions, event-driven autoscaling et `scale to zero`, sans exposer les API Kubernetes natives. Choisir `AKS` lorsque l'équipe a besoin du contrôle Kubernetes et de ses API.

Dans `AKS`, Microsoft gère le control plane et l'équipe paie et administre les worker nodes, généralement basés sur des VMSS. Les mises à jour de Kubernetes remplacent progressivement les nœuds et déplacent les pods ; planifier la capacité et les règles de disruption. Pour administrer le cluster avec `kubectl`, combiner l'intégration Microsoft Entra ID et les autorisations Kubernetes ou `Azure RBAC for Kubernetes` selon le modèle retenu.

## App Service plans

Un `App Service plan` définit la région, l'OS, le tier, la taille et le nombre d'instances. Les applications et slots d'un même plan partagent les instances sous-jacentes et la facturation du plan.

| Tiers | Modèle et points d'examen |
|---|---|
| `Free` / `Shared` | Compute partagé, quotas CPU, pas de scale-out ; test et démonstration |
| `Basic` | Compute dédié, scale-up et scale-out manuel selon les limites du tier, sans autoscale métrique |
| `Standard` | Compute dédié, autoscale, deployment slots, domaines personnalisés et sauvegardes selon les fonctionnalités disponibles |
| `Premium` / `PremiumV2` / `PremiumV3` / `PremiumV4` | Plus de capacité, slots et options de scaling avancées selon le SKU et la région |
| `Isolated` / `IsolatedV2` | App Service Environment, compute et réseau isolés |

`Consumption` est un plan de facturation principalement associé à `Azure Functions`, avec scaling automatique et scale-to-zero ; ne pas le confondre avec un tier d'App Service web classique.

- Plusieurs apps peuvent partager un plan pour réduire le coût, mais elles partagent aussi CPU, mémoire et scaling.
- Isoler une application gourmande, une application d'une autre région ou une application qui doit scaler indépendamment dans un autre plan.
- Une `App Service` héberge l'application ; plusieurs apps peuvent partager un plan.
- `Scale up` change le tier/size ; `Scale out` change le nombre d'instances.
- Le scale-out peut être manuel, `rule-based autoscale` avec metrics/schedules, ou `automatic scaling (elastic scale)` sur les tiers Premium compatibles. Utiliser une moyenne et une fenêtre temporelle pour éviter de réagir à un pic momentané.

## Configurer App Service

- Les sources de déploiement incluent GitHub, Azure DevOps, un container registry, FTP et Local Git selon le scénario.
- `Deployment slots` sont disponibles dans les tiers `Standard`, `Premium` et `Isolated`. Chaque slot est une application active avec son propre hostname ; les slots consomment les ressources du même plan.
- `Deployment slots` servent au staging, aux tests et au swap. Les paramètres `Deployment slot setting` restent liés au slot.
- Un swap staging -> production permet validation, warm-up et rollback par un second swap. Les versions de runtime, `App settings` et `connection strings` peuvent être échangées, sauf si elles sont marquées `Deployment slot setting`. Les identités managées, l'intégration VNet, les custom domains, les certificats, les restrictions IP, `Always On` et les paramètres de scale restent généralement attachés au slot.
- Connaître `TLS/SSL`, `custom domains`, `managed certificates`, `App Service authentication`, `application settings`, `connection strings`, logs et backup.
- Pour un custom domain comme `www.contoso.com`, la première étape est souvent la création d'un enregistrement `TXT` nommé `asuid` contenant l'identifiant de validation fourni par Azure. Une fois vérifié, on configure ensuite le `CNAME` ou l'enregistrement `A` selon le cas.
- `VNet integration` sert au trafic sortant ; `Private Endpoint` sert à l'accès privé entrant.
- `Managed identity` permet à l'application d'accéder à Key Vault, Storage ou d'autres services sans secret dans le code.
- Le `runtime stack` définit le langage/SDK. `Always On` garde l'application chargée et est requis pour les continuous/CRON WebJobs. `HTTPS Only` redirige le trafic HTTP vers HTTPS.
- `Session affinity` (ARR affinity) maintient un client sur la même instance pendant la session.
- `App Service authentication`/`Easy Auth` valide les tokens, gère la session et injecte l'identité avec peu ou pas de code. `Require authentication` redirige les requêtes anonymes vers le fournisseur d'identité.
- Les sources CI/CD incluent GitHub, Bitbucket, local Git et Azure Repos. Pour GitHub, connaître `GitHub Actions` (par défaut) et `App Service Build Service`.
- `Custom domains` : `CNAME` pour un sous-domaine ; `A record` pour l'apex si le registrar refuse CNAME au root.
- `Backup and restore` est disponible à partir de Basic ; au tier Basic, seule la production est sauvegardable. Une restauration complète remplace le contenu et supprime les fichiers absents de la sauvegarde.
- Reconnaître `Health check`, `Deployment Center` et `Application Insights`, qui suit requests, response times, failures, exceptions et custom business events.

### À retenir pour la conception et le diagnostic

- Pour déployer un template ARM via PowerShell, la commande de référence est `New-AzResourceGroupDeployment` avec le paramètre `-ResourceGroupName` ; le `what-if` et le bonne portée du déploiement sont essentiels.
- `Azure Disk Encryption` est la bonne solution quand un scénario exige de chiffrer les disques d'OS et de données avec une clé stockée dans `Azure Key Vault`, tout en gardant le chiffrement même si le disque est exporté ou téléchargé.
- Les règles d'`autoscale` App Service évaluent un seuil sur une fenêtre de temps, puis appliquent un `cooldown` avant une nouvelle action. Une règle peut créer 2 instances après 15 minutes si le CPU reste au-dessus du seuil.
- `Autoscale` est une capacité Azure Monitor basée sur des métriques, des horaires ou une combinaison des deux. Il faut définir une capacité minimale, maximale et par défaut cohérente ; un minimum égal au maximum empêche tout ajustement dynamique.
- Un `VMSS` convient aux workloads identiques avec autoscale ; un simple `VM` reste plus flexible pour un besoin unique ou personnalisé.

## Lab

1. Choisir entre VM, VMSS, App Service, ACI et Container Apps pour cinq scénarios courts.
2. Déployer une VM avec managed disk, NSG et `Boot diagnostics`, puis justifier sa taille.
3. Créer un template Bicep et l'exécuter avec `what-if`.
4. Publier une image dans ACR et la déployer dans ACI ou Container Apps.
5. Créer une App Service, un slot `staging`, puis tester un swap.
6. Configurer un autoscale avec minimum/maximum différents et vérifier l'effet du scale-out et du scale-in.
7. Déployer un container group ACI avec `restartPolicy`, volume Azure Files et accès réseau adapté.
8. Tester `Always On`, `HTTPS Only`, Easy Auth et Application Insights.

## Ressources

- [ARM templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Bicep what-if deployments](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/deploy-what-if)
- [Azure Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- [VM availability options](https://learn.microsoft.com/en-us/azure/virtual-machines/availability)
- [Azure Spot Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms)
- [Availability sets](https://learn.microsoft.com/en-us/azure/virtual-machines/availability-set-overview)
- [VM quotas](https://learn.microsoft.com/en-us/azure/virtual-machines/quotas)
- [Azure App Service plans](https://learn.microsoft.com/en-us/azure/app-service/overview-hosting-plans)
- [App Service deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots)
- [Configure a custom domain for Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/app-service-web-tutorial-custom-domain)
- [Azure Container Instances overview](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-overview)
- [ACI restart policy](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-restart-policy)
- [ACI Azure Files volumes](https://learn.microsoft.com/en-us/azure/container-instances/container-instances-volume-azure-files)
- [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/)
- [App Service backup](https://learn.microsoft.com/en-us/azure/app-service/manage-backup)
- [Azure Container Registry SKUs](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-skus)