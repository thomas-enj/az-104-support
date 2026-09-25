# Module 01 - Manage Azure identities and governance

[Retour au README](../README.md) | [Module suivant : Storage](02-storage.md)

## Vue d’ensemble
Ce module couvre les fondations de l’identité Azure, la gouvernance des ressources, le contrôle d’accès et la conformité. Il est central pour l’examen AZ-104 car il regroupe les décisions de conception, les permissions et les mécanismes de protection de l’environnement.

## Plan du module
1. Microsoft Entra ID, utilisateurs, groupes, appareils et licences
2. SSPR, Conditional Access et délégation par Administrative Unit
3. Azure RBAC, rôles intégrés et rôles personnalisés
4. Gouvernance, management groups, Azure Policy, tags et locks
5. Labs, cas de validation et pièges fréquents

## Microsoft Entra ID

`Microsoft Entra ID` fournit l'identité cloud, l'authentification et le contrôle d'accès. Il ne remplace pas automatiquement `Active Directory Domain Services (AD DS)`, qui fournit notamment LDAP, Kerberos, NTLM et `Group Policy`.

La première décision consiste à distinguer l'identité cloud (`Microsoft Entra ID`) de l'annuaire de domaine traditionnel (`AD DS`). `Entra Connect Sync` et `Cloud Sync` synchronisent des identités existantes ; ils ne transforment pas Entra ID en contrôleur de domaine.

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
| `P1` | `Conditional Access`, groupes dynamiques et SSPR avec password writeback |
| `P2` | P1 + `Identity Protection`, stratégies Conditional Access basées sur le risque et `Privileged Identity Management (PIM)` |

Les fonctionnalités peuvent aussi être incluses dans des suites Microsoft 365 ou Enterprise Mobility + Security. Vérifier le plan exact avant de conclure qu'une fonction est disponible.

### Utilisateurs, groupes et appareils

- Une identité peut être cloud-only, synchronisée depuis AD DS ou `Guest`/B2B pour un utilisateur externe.
- Un utilisateur supprimé reste dans `Deleted users` pendant 30 jours avant la suppression définitive. Durant cette période, il peut être restauré avec ses appartenances et ses permissions récupérables.
- `UserType = Member` correspond généralement à un utilisateur interne ; `UserType = Guest` à une identité B2B externe. Un invité peut être converti en membre sans recréer l'objet lorsque le scénario l'exige.
- Un `Security group` contrôle l'accès ; un `Microsoft 365 group` fournit des fonctions de collaboration comme la boîte aux lettres, le calendrier et SharePoint.
- `Assigned membership` est gérée manuellement ; `Dynamic membership` évalue des attributs et nécessite généralement P1.
- Une règle de `Dynamic membership` doit être évaluée dans le contexte de l'objet concerné : utilisateurs et appareils n'exposent pas exactement les mêmes attributs. Vérifier la syntaxe de la règle et son mode d'évaluation avant d'utiliser le groupe pour une licence ou une policy.
- `Group-based licensing` attribue et retire automatiquement les licences. Renseigner l'`Usage location` avant l'attribution et compter les membres uniques lorsqu'un utilisateur appartient à plusieurs groupes licenciés.
- L'attribution peut échouer si le tenant n'a plus de licences disponibles ou si des plans de services mutuellement exclusifs sont activés. Le statut d'erreur de licence doit être contrôlé au niveau du groupe et de l'utilisateur.
- `Custom security attributes` sont des paires clé-valeur propres à l'organisation pour classifier des objets et affiner certains contrôles. Pour un scénario de création et d'attribution, il faut souvent lier la définition et l'assignation à des rôles dédiés : `Attribute Definition Administrator` et `Attribute Assignment Administrator`.
- `Microsoft Entra registered` correspond typiquement à un appareil BYOD ; `Microsoft Entra joined` à un appareil cloud-only ; `Microsoft Entra hybrid joined` conserve la jonction AD DS et ajoute l'identité Entra. `Microsoft Intune` applique les stratégies MDM et de conformité.

En pratique, si un examen demande de créer un attribut personnalisé puis l'attribuer à un utilisateur, il faut regarder à la fois `Custom security attributes` et `Roles and administrators`. Le rôle `User Administrator` n'est pas suffisant pour définir ou assigner ces attributs sans les rôles d'administration dédiés.

La documentation actuelle précise même que `Global Administrator` ne dispose pas automatiquement des permissions de lecture, de définition ou d'assignation de ces attributs. Le contrôle se fait dans les rôles Entra dédiés : `Attribute Definition Administrator` pour définir les attributs, `Attribute Assignment Administrator` pour les affecter et `Attribute Assignment Reader` pour les lire.

### Synchronisation hybride

- `Microsoft Entra Cloud Sync` utilise un agent léger local et un service de provisioning géré dans le cloud. Il convient notamment aux forêts multiples ou déconnectées.
- `Microsoft Entra Connect Sync` reste adapté aux scénarios nécessitant davantage de fonctionnalités de synchronisation et de transformation locales. Cloud Sync n'est pas un remplacement universel.
- `SCIM 2.0` sert surtout au provisioning d'applications compatibles ; une source RH sans endpoint SCIM peut utiliser `API-driven inbound provisioning`.
- Le `password writeback` et SSPR nécessitent la configuration adaptée de la synchronisation et du connecteur local.

Pour le provisioning, distinguer `SCIM 2.0`, `Microsoft Entra provisioning service`, `API-driven inbound provisioning`, `Dynamic groups` et `Group-based licensing`. Pour chaque membre unique d'un groupe sous licence, une licence disponible doit être possédée ; 1 000 membres uniques nécessitent donc au moins 1 000 licences correspondantes. Vérifier les éditions et licences exactes dans la documentation actuelle.

### Conditional Access

`Conditional Access` évalue l'utilisateur, l'application, la localisation, l'appareil et le risque, puis applique `Grant access`, `Require multifactor authentication` ou `Block access`.

Réflexes : tester d'abord en `Report-only`, prévoir des `emergency access accounts` et vérifier les exclusions pour éviter un lockout administratif. `Identity Protection` est le choix lorsqu'un scénario parle de risque de connexion ou de risque utilisateur.

### Administrative units et délégation

Une `Administrative unit (AU)` limite la portée de certains rôles Entra à un sous-ensemble d'utilisateurs, de groupes ou d'appareils. Elle ne remplace ni un `resource group` Azure ni un groupe de sécurité utilisé pour l'accès aux ressources. Pour une délégation locale, combiner l'AU avec un rôle Entra approprié et vérifier si le rôle est assigné sur l'AU plutôt qu'au niveau du tenant.

Les objets d'une AU peuvent être ajoutés manuellement ou selon une règle dynamique. Une AU ne remplace pas un groupe de sécurité : elle définit principalement la portée administrative d'un rôle.

## Azure RBAC

Les rôles Microsoft Entra ID administrent le tenant et ses objets d'annuaire ; les rôles Azure RBAC administrent les ressources Azure. Un `Global Administrator` ne reçoit pas automatiquement les droits sur les VM, réseaux ou comptes de stockage. Pour obtenir temporairement ce contrôle, il doit activer `Access management for Azure resources`, ce qui lui attribue temporairement `User Access Administrator` au niveau racine de la subscription.

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

`NotActions` ne constitue pas un refus explicite : une permission peut encore être accordée par un autre rôle. Pour analyser un scénario, additionner les rôles hérités et vérifier séparément les `DataActions`, les conditions et les `deny assignments`.

### Rôles personnalisés et structure JSON

Créer un `custom role` uniquement lorsqu'aucun rôle intégré ne respecte le principe du moindre privilège. Les propriétés principales sont `Actions`, `NotActions`, `DataActions`, `NotDataActions` et `AssignableScopes`.

```json
{
	"Name": "Support VM Custom",
	"IsCustom": true,
	"Description": "Peut redémarrer les VM sans les supprimer.",
	"Actions": [
		"Microsoft.Compute/virtualMachines/read",
		"Microsoft.Compute/virtualMachines/start/action",
		"Microsoft.Compute/virtualMachines/restart/action"
	],
	"NotActions": [
		"Microsoft.Compute/virtualMachines/delete"
	],
	"DataActions": [],
	"NotDataActions": [],
	"AssignableScopes": [
		"/subscriptions/<subscription-id>"
	]
}
```

`Actions` concerne le control plane ; `DataActions` concerne le contenu des données. `AssignableScopes` détermine où le rôle peut être attribué. La création, la modification ou la suppression d'un rôle personnalisé nécessite `Microsoft.Authorization/roleDefinitions/write`, généralement détenu par `Owner` ou `User Access Administrator`. Un tenant Microsoft Entra peut contenir au maximum 5 000 rôles personnalisés.

Pour déplacer une ressource entre deux resource groups, l'appelant doit disposer des permissions de déplacement sur le scope source et sur le scope destination, ainsi que des permissions nécessaires sur les ressources concernées.

## Governance, subscriptions et architecture

- Hiérarchie de gouvernance : `Tenant root group -> Management groups -> Subscriptions -> Resource groups -> Resources`.
- Un tenant possède un seul `Tenant root group`. Les management groups peuvent être imbriqués jusqu'à six niveaux de profondeur, hors racine et subscription, et les assignments RBAC/Policy héritent vers les descendants.
- Un tenant peut contenir jusqu'à 10 000 management groups. La profondeur maximale de la hiérarchie est de six niveaux, hors tenant root group et subscriptions.
- Un `resource group` regroupe des ressources ayant un cycle de vie commun. Une ressource n'appartient qu'à un resource group et celui-ci ne peut pas être renommé.
- Un resource group peut contenir des ressources de plusieurs régions ; ses métadonnées sont stockées dans une région.
- Supprimer un resource group supprime ses ressources.
- Une subscription est une frontière de `billing` et d'`access control` et est associée à un seul Microsoft Entra directory à un instant donné.
- Les `management groups` organisent les subscriptions et permettent l'héritage de RBAC et de Policy.
- `Tags` servent au coût, propriétaire et environnement ; ils ne sont pas automatiquement hérités par toutes les ressources.
- Une ressource peut recevoir jusqu'à 50 tags. Pour appliquer automatiquement les tags d'un resource group à ses ressources, utiliser Azure Policy avec l'effet `Modify` ou `Append`.
- `CanNotDelete` bloque la suppression ; `ReadOnly` bloque les modifications. Un lock s'applique aussi aux enfants et peut bloquer une opération de gestion légitime.
- `Cost Management` fournit `Cost analysis`, `Budgets` et alertes de coût. `Azure Advisor` fournit des recommandations de coût, sécurité, fiabilité, performance et excellence opérationnelle.
- Une `Azure region` contient un ou plusieurs datacenters reliés par un réseau à faible latence. Une région qui prend en charge les `Availability Zones` en propose généralement au moins trois ; vérifier les services réellement zonaux.
- Une `region pair` aide la résilience entre deux régions d'une même geography.
- Une `sovereign region` répond à des exigences d'isolation, de résidence ou de conformité.
- Un service `zonal` est placé dans une zone ; un service `zone-redundant` réplique entre zones ; un service `non-regional` n'est pas déployé dans une région choisie.
- `Microsoft Entra ID`, `Azure DNS` et `Azure Traffic Manager` sont des exemples de services globaux/non-regional ; une VM est une ressource régionale.
- Les `management groups` peuvent contenir des subscriptions et d'autres management groups et permettent d'appliquer une policy sans répéter l'assignment à chaque subscription.

Les `tags` facilitent le suivi des coûts, des propriétaires et des environnements, mais ils ne sont pas automatiquement hérités par toutes les ressources. Une `policy` peut imposer leur présence ou leur valeur. Les `locks` protègent le control plane : `CanNotDelete` bloque la suppression et `ReadOnly` bloque aussi les modifications, y compris pour un `Owner`.

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

Une policy `Deny` bloque les nouvelles créations ou modifications non conformes ; elle ne supprime pas automatiquement les ressources déjà présentes. Les ressources existantes peuvent être signalées comme non conformes et nécessiter une `remediation task` selon l'effet utilisé.

Une `Policy assignment` associe la policy ou l'initiative à un scope et une `Exemption` documente une exception ciblée. Azure Policy agit sur le `control plane` pour évaluer l'état des ressources et n'est pas un mécanisme de filtrage des données applicatives.

**RBAC vs Policy :** RBAC répond à « qui peut agir ? » ; Policy répond à « quelles configurations sont autorisées ou conformes ? ». Une policy `Deny` peut bloquer une création même si l'utilisateur est `Owner`.

Pour tester une policy en production sans bloquer les déploiements, utiliser `enforcementMode: DoNotEnforce` ou un effet `Audit`, puis examiner l'impact avant d'activer `Deny`.

## SSPR

SSPR (Self-Service Password Reset) permet à l'utilisateur de réinitialiser son mot de passe sans help desk. Distinguer le changement par un utilisateur `signed-in` du reset par un utilisateur `signed-out`.

- Déploiement : `None`, `Selected` ou `All`.
- `Selected` est adapté à un pilote : la stratégie est appliquée aux membres d'un groupe de sécurité sélectionné. Pour les comptes administrateurs, prévoir généralement deux méthodes d'authentification.
- `Authentication methods` et `Number of methods required to reset` sont configurables ; l'utilisateur doit avoir enregistré le nombre minimal de méthodes.
- Les méthodes peuvent inclure `Microsoft Authenticator`, code ou notification mobile, `Email OTP`, téléphone et questions de sécurité selon la policy. Configurer une ou deux méthodes requises ; éviter de faire des questions de sécurité ou du SMS le seul facteur.
- Les questions de sécurité nécessitent plusieurs réponses lors de l'inscription et de la réinitialisation ; elles sont moins robustes qu'une application d'authentification. Pour cibler les invités dans un groupe dynamique, utiliser l'attribut `user.userType` avec la valeur `Guest` plutôt qu'un simple filtre sur le domaine de messagerie.
- SSPR est disponible avec Microsoft Entra ID Free ; `Conditional Access` exige généralement P1, tandis que le `password writeback` et les stratégies de risque nécessitent une capacité Premium adaptée.
- Les comptes administrateurs utilisent une authentification renforcée, généralement deux méthodes.
- `Security questions` sont moins recommandées ; SMS présente des risques de fraude.
- `CAPTCHA` vérifie que la demande vient d'un humain.
- `Password writeback` renvoie le changement vers l'AD on-premises.
- `Notify all admins when other admins reset their password` aide à détecter une activité suspecte.
- Le rôle minimal de configuration est à vérifier dans la documentation Entra actuelle ; `Authentication Policy Administrator` est le rôle à rechercher selon la configuration.
- SSPR concerne le reset d'un utilisateur qui ne peut plus se connecter ; un utilisateur déjà connecté peut changer son mot de passe sans utiliser SSPR. Pour les identités synchronisées, le `password writeback` permet de répercuter le nouveau mot de passe vers AD DS.

### À retenir pour l'examen

- Le `Co-administrator` se place au niveau de la `subscription` ; il ne s'applique pas à un resource group ou à une VM isolée.
- `Owner` et `Contributor` sont des rôles RBAC modernes, alors que le `Co-administrator` correspond à un ancien modèle de gestion abonnement-level.
- Une `Policy` `Deny` peut bloquer une création même si l'utilisateur est `Owner` ; le contrôle d'accès et la conformité sont deux mécanismes distincts.
- Un `resource guard` est requis pour activer la `Multi-User Authorization (MAU)` sur un `Recovery Services vault`.
- `Management group` n'est pas un remplacement de `resource group` ; il sert à l'organisation, à l'héritage de RBAC/Policy et à la gouvernance globale.

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
- [Manage access to custom security attributes](https://learn.microsoft.com/en-us/entra/fundamentals/custom-security-attributes-manage)
- [Microsoft Entra built-in roles](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/permissions-reference)
- [Dynamic membership rules](https://learn.microsoft.com/en-us/entra/identity/users/groups-dynamic-membership)
- [Administrative units](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/administrative-units)
- [Azure management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)
- [Azure RBAC overview](https://learn.microsoft.com/en-us/azure/role-based-access-control/overview)
- [Azure Policy overview](https://learn.microsoft.com/en-us/azure/governance/policy/overview)
- [SSPR deep dive](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-sspr-howitworks)