# Module 03 - Deploy and manage Azure compute resources

[Retour au README](../README.md) | [Module suivant : Virtual networking](04-virtual-networking.md)

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

## Virtual machines

Lors de la création, relier `Image`, `VM size`, `Disks`, `NIC`, `VNet/Subnet`, `Public IP`, `NSG`, authentification et `Availability options`.

- `Managed disks` : `Premium SSD`, `Standard SSD`, `Standard HDD`, `Ultra Disk`.
- Distinguer `OS disk` et `data disks`.
- Le choix d'une `VM size` dépend des vCPU, mémoire, IOPS, débit disque/réseau, région et quotas.
- Un `snapshot` est une copie ponctuelle ; une `disk image` sert à créer des VM similaires.
- `Availability Zones` séparent les instances dans des zones physiques distinctes.
- `Availability Sets` utilisent `fault domains` et `update domains` dans un datacenter.
- `Virtual Machine Scale Sets (VMSS)` gèrent des VM identiques avec autoscaling.
- `Encryption at host` chiffre données temporaires et caches du host, en complément des managed disks.
- Une VM peut être déplacée entre resource groups/subscriptions sous conditions ; changer de VNet implique généralement de la recréer en conservant les disques.

### VMSS et autoscale

- `Scale out` ajoute des instances ; `Scale in` en retire. Prévoir les deux règles pour absorber la charge puis réduire les coûts.
- Définir `minimum`, `maximum` et `default instance count` avec une marge réelle ; si minimum et maximum sont identiques, aucun scaling dynamique n'est possible.
- `Scale-in policy` détermine quelles instances sont retirées. `Max spreading` répartit les instances sur autant de fault domains que possible lorsque la résilience est prioritaire.
- Les VMSS supportent l'autoscale métrique et planifié. Les `Availability Zones` et les `region pairs` répondent à des niveaux de panne différents.
- Une `Availability Set` protège contre hardware/réseau et mises à jour dans un datacenter, pas contre un bug OS ou applicatif ; prévoir Backup/DR séparément.

Diagnostic Portal : `Boot diagnostics`, `Run command`, `Activity log`, `Metrics`, `Disks` et `Networking > Effective security rules`.

## Containers

| Service | Choix adapté |
|---|---|
| `Azure Container Instances (ACI)` | Containers ponctuels, sans orchestration complexe |
| `Azure Container Apps` | Applications containerisées, ingress, revisions et scaling serverless |
| `Azure Container Registry (ACR)` | Registry privé d'images et artefacts |
| `Azure Kubernetes Service (AKS)` | Orchestration Kubernetes |

Dans ACR, connaître `repositories`, `tags`, `admin user` et l'authentification par Entra/managed identity.

### Azure Container Instances

- `ACI` convient à un job court, un démarrage rapide et une exposition directe par IP/FQDN sans gérer de cluster.
- Un `container group` partage le cycle de vie, le réseau local et le port namespace. Deux containers du même groupe ne doivent pas écouter le même port ; changer le port d'écoute du sidecar.
- Le pattern multi-container convient à un sidecar de monitoring qui observe le container principal.
- L'IP publique et le FQDN d'un container group sont libérés à sa suppression et ne sont pas garantis lors d'un nouveau déploiement.
- Les VM offrent une boundary d'isolation plus forte que les containers pour des workloads de tenants strictement séparés.

### Azure Container Apps et AKS

`Azure Container Apps` est une plateforme serverless construite sur Kubernetes qui abstrait l'infrastructure, fournit ingress, revisions, event-driven autoscaling et `scale to zero`, sans exposer les API Kubernetes natives. Choisir `AKS` lorsque l'équipe a besoin du contrôle Kubernetes et de ses API.

## App Service

- `App Service plan` définit région, OS, pricing tier, taille et nombre de VM instances. Plusieurs apps partagent les ressources et la facturation du plan ; isoler une application très consommatrice si elle doit scaler indépendamment.
- `Free` et `Shared` utilisent du shared compute pour le développement/test. `Basic`, `Standard` et `Premium` utilisent du dedicated compute. `Isolated`/`IsolatedV2` ajoutent des VM dédiées et une network isolation.
- Une `App Service` héberge l'application ; plusieurs apps peuvent partager un plan.
- `Scale up` change le tier/size ; `Scale out` change le nombre d'instances.
- Le scale-out peut être manuel, `rule-based autoscale` avec metrics/schedules, ou `automatic scaling (elastic scale)` sur les tiers Premium compatibles. Utiliser une moyenne et une fenêtre temporelle pour éviter de réagir à un pic momentané.
- `Standard` fournit notamment des deployment slots et rule-based autoscale ; `PremiumV2/PremiumV3` ajoutent l'elastic scale selon la disponibilité actuelle. `IsolatedV2` vise l'isolation et la grande capacité.
- `Deployment slots` servent au staging, aux tests et au swap. Les paramètres `Deployment slot setting` restent liés au slot.
- Un swap staging -> production permet validation, warm-up et rollback par un second swap. `App settings`, `connection strings` et runtime settings peuvent être échangés ; custom domains, TLS/SSL certificates et certains scaling settings restent attachés au slot.
- Connaître `TLS/SSL`, `custom domains`, `managed certificates`, `App Service authentication`, `application settings`, `connection strings`, logs et backup.
- `VNet integration` sert au trafic sortant ; `Private Endpoint` sert à l'accès privé entrant.
- Le `runtime stack` définit le langage/SDK. `Always On` garde l'application chargée et est requis pour les continuous/CRON WebJobs. `HTTPS Only` redirige le trafic HTTP vers HTTPS.
- `Session affinity` (ARR affinity) maintient un client sur la même instance pendant la session.
- `App Service authentication`/`Easy Auth` valide les tokens, gère la session et injecte l'identité avec peu ou pas de code. `Require authentication` redirige les requêtes anonymes vers le fournisseur d'identité.
- Les sources CI/CD incluent GitHub, Bitbucket, local Git et Azure Repos. Pour GitHub, connaître `GitHub Actions` (par défaut) et `App Service Build Service`.
- `Custom domains` : `CNAME` pour un sous-domaine ; `A record` pour l'apex si le registrar refuse CNAME au root.
- `Backup and restore` est disponible à partir de Basic ; au tier Basic, seule la production est sauvegardable. Une restauration complète remplace le contenu et supprime les fichiers absents de la sauvegarde.
- Reconnaître `Health check`, `Deployment Center` et `Application Insights`, qui suit requests, response times, failures, exceptions et custom business events.

## Lab

1. Déployer une VM avec managed disk, NSG et `Boot diagnostics`.
2. Créer un template Bicep et l'exécuter avec `what-if`.
3. Publier une image dans ACR et la déployer dans ACI ou Container Apps.
4. Créer une App Service, un slot `staging`, puis tester un swap.
5. Configurer un autoscale avec minimum/maximum différents et tester `Always On`, `HTTPS Only`, Easy Auth et Application Insights.

## Ressources

- [ARM templates](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/)
- [Bicep documentation](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/)
- [Azure Virtual Machines](https://learn.microsoft.com/en-us/azure/virtual-machines/)
- [Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/)