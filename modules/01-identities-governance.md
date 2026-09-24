# Module 01 - Manage Azure identities and governance

[Retour au README](../README.md) | [Module suivant : Storage](02-storage.md)

## Microsoft Entra ID

`Microsoft Entra ID` fournit l'identité cloud, l'authentification et le contrôle d'accès. Il ne remplace pas automatiquement `Active Directory Domain Services (AD DS)`, qui fournit notamment LDAP, Kerberos, NTLM et `Group Policy`.

À savoir faire dans `Microsoft Entra ID` :

- Créer et modifier `Users` et `Groups`.
- Distinguer les groupes `Security`, `Microsoft 365`, `Assigned` et `Dynamic`.
- Gérer les `Guest users`, `External identities` et les `Licenses`.
- Configurer `Self-service password reset (SSPR)`.
- Différencier user, group, `service principal` et `managed identity`.

| Besoin | Solution |
|---|---|
| Identités cloud, SSO SaaS et authentification moderne | `Microsoft Entra ID` |
| Application legacy nécessitant LDAP, Kerberos, NTLM ou GPO | `Microsoft Entra Domain Services` |
| Synchroniser plusieurs forests avec peu d'infrastructure | `Microsoft Entra Cloud Sync` |
| Synchronisation riche avec agent local | `Microsoft Entra Connect Sync` |
| Inviter un consultant avec son propre compte | `External ID` / `Guest` B2B |
| Conserver GPO et ajouter Entra SSO | `Microsoft Entra hybrid joined` |
| Appareil cloud-only | `Microsoft Entra joined` |
| Appareil personnel BYOD | `Microsoft Entra registered` |
| MFA, pays, appareil et conditions de connexion | `Conditional Access`, généralement P1 |
| `risky sign-in`, `impossible travel`, `user risk` | `Identity Protection`, généralement P2 |
| Élévation temporaire avec approbation et journalisation | `Privileged Identity Management (PIM)`, généralement P2 |

`Microsoft Entra ID` n'a pas d'OU ni de GPO. Pour la configuration des appareils, utiliser notamment `Microsoft Intune` et `Conditional Access`.

### Éditions et licences

| Édition | Fonctions à retenir pour l'AZ-104 |
|---|---|
| `Free` | Identités cloud, SSO de base et fonctions fondamentales |
| `P1` | `Conditional Access`, groupes dynamiques et SSPR pour les utilisateurs |
| `P2` | P1 + `Identity Protection` et `Privileged Identity Management (PIM)` |

Les fonctionnalités peuvent aussi être incluses dans des suites Microsoft 365 ou Enterprise Mobility + Security. Vérifier le plan exact avant de conclure qu'une fonction est disponible.

### Utilisateurs, groupes et appareils

- Une identité peut être cloud-only, synchronisée depuis AD DS ou `Guest`/B2B pour un utilisateur externe.
- Un `Security group` contrôle l'accès ; un `Microsoft 365 group` fournit des fonctions de collaboration comme la boîte aux lettres, le calendrier et SharePoint.
- `Assigned membership` est gérée manuellement ; `Dynamic membership` évalue des attributs et nécessite généralement P1.
- `Group-based licensing` attribue et retire automatiquement les licences. Renseigner l'`Usage location` avant l'attribution et compter les membres uniques lorsqu'un utilisateur appartient à plusieurs groupes licenciés.
- `Custom security attributes` sont des paires clé-valeur propres à l'organisation pour classifier des objets et affiner certains contrôles.
- `Microsoft Entra registered` correspond typiquement à un appareil BYOD ; `Microsoft Entra joined` à un appareil cloud-only ; `Microsoft Entra hybrid joined` conserve la jonction AD DS et ajoute l'identité Entra. `Microsoft Intune` applique les stratégies MDM et de conformité.

### Synchronisation hybride

- `Microsoft Entra Cloud Sync` utilise un agent léger local et un service de provisioning géré dans le cloud. Il convient notamment aux forêts multiples ou déconnectées.
- `Microsoft Entra Connect Sync` reste adapté aux scénarios nécessitant davantage de fonctionnalités de synchronisation et de transformation locales. Cloud Sync n'est pas un remplacement universel.
- `SCIM 2.0` sert surtout au provisioning d'applications compatibles ; une source RH sans endpoint SCIM peut utiliser `API-driven inbound provisioning`.
- Le `password writeback` et SSPR nécessitent la configuration adaptée de la synchronisation et du connecteur local.

Pour le provisioning, distinguer `SCIM 2.0`, `Microsoft Entra provisioning service`, `API-driven inbound provisioning`, `Dynamic groups` et `Group-based licensing`. Pour chaque membre unique d'un groupe sous licence, une licence disponible doit être possédée ; 1 000 membres uniques nécessitent donc au moins 1 000 licences correspondantes. Vérifier les éditions et licences exactes dans la documentation actuelle.

### Conditional Access

`Conditional Access` évalue l'utilisateur, l'application, la localisation, l'appareil et le risque, puis applique `Grant access`, `Require multifactor authentication` ou `Block access`.

Réflexes : tester d'abord en `Report-only`, prévoir des `emergency access accounts` et vérifier les exclusions pour éviter un lockout administratif. `Identity Protection` est le choix lorsqu'un scénario parle de risque de connexion ou de risque utilisateur.

## Azure RBAC

Une `role assignment` contient :

```text
Security principal + Role definition + Scope
```

Scopes, du plus large au plus précis :

```text
Management group -> Subscription -> Resource group -> Resource
```

Un rôle à un scope parent est hérité par les enfants. Les permissions sont additives ; les `deny assignments` et conditions peuvent bloquer l'accès.

| Rôle | À retenir |
|---|---|
| `Owner` | Gestion complète, y compris les role assignments |
| `Contributor` | Gestion des ressources, sans role assignments |
| `Reader` | Lecture seule |
| `User Access Administrator` | Gestion des accès |
| `Virtual Machine Contributor` | Gestion des VM |
| `Network Contributor` | Gestion des ressources réseau |
| `Storage Account Contributor` | Gestion du compte, pas forcément des blobs |
| `Storage Blob Data Reader/Contributor/Owner` | Permissions data plane Blob |

`Storage Account Contributor` et `Storage Blob Data Reader` ne donnent donc pas le même accès. Les role assignments se gèrent dans `Access control (IAM)`.

`Actions - NotActions` calcule les permissions de management ; `DataActions - NotDataActions` suit le même principe pour le data plane. Si aucun built-in role ne convient, créer un `custom role` minimal.

## Governance, subscriptions et architecture

- Hiérarchie de gouvernance : `Tenant root group -> Management groups -> Subscriptions -> Resource groups -> Resources`.
- Un tenant possède un seul `Tenant root group`. Les management groups peuvent être imbriqués jusqu'à six niveaux de profondeur, hors racine et subscription, et les assignments RBAC/Policy héritent vers les descendants.
- Un `resource group` regroupe des ressources ayant un cycle de vie commun. Une ressource n'appartient qu'à un resource group et celui-ci ne peut pas être renommé.
- Un resource group peut contenir des ressources de plusieurs régions ; ses métadonnées sont stockées dans une région.
- Supprimer un resource group supprime ses ressources.
- Une subscription est une frontière de `billing` et d'`access control` et est associée à un seul Microsoft Entra directory à un instant donné.
- Les `management groups` organisent les subscriptions et permettent l'héritage de RBAC et de Policy.
- `Tags` servent au coût, propriétaire et environnement ; ils ne sont pas automatiquement hérités par toutes les ressources.
- `CanNotDelete` bloque la suppression ; `ReadOnly` bloque les modifications. Un lock s'applique aussi aux enfants et peut bloquer une opération de gestion légitime.
- `Cost Management` fournit `Cost analysis`, `Budgets` et alertes de coût. `Azure Advisor` fournit des recommandations de coût, sécurité, fiabilité, performance et excellence opérationnelle.
- Une `Azure region` contient un ou plusieurs datacenters reliés par un réseau à faible latence. Une région qui prend en charge les `Availability Zones` en propose généralement au moins trois ; vérifier les services réellement zonaux.
- Une `region pair` aide la résilience entre deux régions d'une même geography.
- Une `sovereign region` répond à des exigences d'isolation, de résidence ou de conformité.
- Un service `zonal` est placé dans une zone ; un service `zone-redundant` réplique entre zones ; un service `non-regional` n'est pas déployé dans une région choisie.
- `Microsoft Entra ID`, `Azure DNS` et `Azure Traffic Manager` sont des exemples de services globaux/non-regional ; une VM est une ressource régionale.
- Les `management groups` peuvent contenir des subscriptions et d'autres management groups et permettent d'appliquer une policy sans répéter l'assignment à chaque subscription.

Pour `development`, `test` et `production`, utiliser des resource groups distincts. Utiliser des subscriptions distinctes lorsque la séparation du billing, des quotas ou des accès doit être forte.

## Azure Policy

| Élément | Usage |
|---|---|
| `Built-in policy` | Définition Microsoft |
| `Custom policy` | Règle spécifique à l'organisation |
| `Initiative` | Groupe de policy definitions |
| `Deny` | Bloque une création ou modification |
| `Audit` | Autorise mais marque non conforme |
| `Append` | Ajoute une propriété au déploiement lorsque la définition le permet |
| `Modify` | Modifie une propriété si les permissions le permettent |
| `DeployIfNotExists` | Déploie une configuration manquante |
| `AuditIfNotExists` | Signale une configuration absente |
| `DoNotEnforce` | Teste sans appliquer l'effet bloquant |

`Greenfield` signifie que la policy précède la ressource ; `Brownfield` concerne les ressources existantes. Le cycle de conformité est généralement de 24 heures ; il peut être déclenché avec `az policy state trigger-scan`.

Une `remediation task` traite les ressources existantes pour `Modify` et `DeployIfNotExists`, si l'identité managée de l'assignment possède les permissions nécessaires. Une `policy exemption` documente une exception ciblée avec les catégories `Waiver` ou `Mitigated`. L'état `Conflicting` indique des assignments contradictoires.

Une ressource est évaluée lors de sa création ou mise à jour, lors d'une nouvelle assignment ou modification de policy, puis lors du cycle périodique d'environ 24 heures. Une policy `Deny` peut bloquer une requête même si l'appelant dispose d'un rôle `Owner` ; RBAC et Policy répondent à des questions différentes.

Une `Policy assignment` associe la policy ou l'initiative à un scope et une `Exemption` documente une exception ciblée. Azure Policy agit sur le `control plane` pour évaluer l'état des ressources et n'est pas un mécanisme de filtrage des données applicatives.

**RBAC vs Policy :** RBAC répond à « qui peut agir ? » ; Policy répond à « quelles configurations sont autorisées ou conformes ? ». Une policy `Deny` peut bloquer une création même si l'utilisateur est `Owner`.

Pour tester une policy en production sans bloquer les déploiements, utiliser `enforcementMode: DoNotEnforce` ou un effet `Audit`, puis examiner l'impact avant d'activer `Deny`.

## SSPR

SSPR permet à l'utilisateur de réinitialiser son mot de passe sans help desk. Distinguer le changement par un utilisateur `signed-in` du reset par un utilisateur `signed-out`.

- Déploiement : `None`, `Selected` ou `All`.
- `Authentication methods` et `Number of methods required to reset` sont configurables ; l'utilisateur doit avoir enregistré le nombre minimal de méthodes.
- Les méthodes peuvent inclure `Microsoft Authenticator`, code ou notification mobile, `Email OTP`, téléphone et questions de sécurité selon la policy. Configurer une ou deux méthodes requises ; éviter de faire des questions de sécurité ou du SMS le seul facteur.
- Les comptes administrateurs utilisent une authentification renforcée, généralement deux méthodes.
- `Security questions` sont moins recommandées ; SMS présente des risques de fraude.
- `CAPTCHA` vérifie que la demande vient d'un humain.
- `Password writeback` renvoie le changement vers l'AD on-premises.
- `Notify all admins when other admins reset their password` aide à détecter une activité suspecte.
- Le rôle minimal de configuration est à vérifier dans la documentation Entra actuelle ; `Authentication Policy Administrator` est le rôle à rechercher selon la configuration.
- SSPR concerne le reset d'un utilisateur qui ne peut plus se connecter ; un utilisateur déjà connecté peut changer son mot de passe sans utiliser SSPR. Pour les identités synchronisées, le `password writeback` permet de répercuter le nouveau mot de passe vers AD DS.

## Lab

1. Créer `rg-az104-lab` et les tags `environment=training` et `owner=<name>`.
2. Assigner `Reader` à un groupe au niveau du resource group.
3. Créer une policy de tag en `Audit`, puis la tester en `Deny`.
4. Vérifier `Access control (IAM) > Check access`.
5. Tester un `CanNotDelete lock`.
6. Configurer un pilote SSPR avec `Selected`.

## Ressources

- [Microsoft Entra overview](https://learn.microsoft.com/en-us/entra/fundamentals/whatis)
- [Microsoft Entra Cloud Sync](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/what-is-cloud-sync)
- [Microsoft Entra licensing](https://learn.microsoft.com/en-us/entra/fundamentals/licensing)
- [Azure management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- [Azure RBAC overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [SSPR deep dive](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sspr-howitworks)