# Le Test de Peter 2026

> Reproduire expérimentalement, avec des composants Linux actuels, l'architecture imaginée dans l'article « Le test de Peter » et déterminer jusqu'où elle reste techniquement réalisable en 2026.

**Statut :** étude expérimentale en préparation
**Langue principale :** français
**Cible :** Linux 2026, virtualisation, initramfs, NFS, OverlayFS et LVM
**Auteur / mainteneur :** `valorisa`

## À propos du projet

En 2017, Étienne Dublé publie dans *GNU/Linux Magazine* n°206 l'article « Le test de Peter ». Le point de départ est une scène de série télévisée qui semble techniquement incohérente : un système paraît démarrer depuis un support temporaire, modifier ou installer son environnement, afficher ensuite le bureau, poursuivre une opération de migration, puis continuer à fonctionner après retrait du support sans second redémarrage visible.

L'article ne se contente pas de conclure que la scène est « impossible » ou qu'elle relève de la magie télévisuelle. Il reconstruit progressivement une architecture Linux susceptible de rendre le comportement plausible, en combinant notamment :

- un `initramfs` ;
- un root filesystem distant via NFS ;
- `exec` et `chroot` pour la transition vers l'init du système ;
- OverlayFS pour superposer une couche d'écriture locale ;
- LVM pour organiser le stockage ;
- `pvmove` pour déplacer les extents physiques ;
- `debootstick` comme exemple concret d'outillage capable de transformer et migrer un environnement Debian.

Près d'une décennie plus tard, les primitives essentielles de cette démonstration existent toujours.

Ce dépôt reprend donc le raisonnement de l'article, mais refuse de considérer la compatibilité conceptuelle de chaque primitive comme une preuve de compatibilité de l'ensemble.

> **Le but n'est pas de démontrer que Peter avait raison. Le but est de découvrir expérimentalement jusqu'où son raisonnement tient encore.**

## Question centrale

La question expérimentale est la suivante :

> Peut-on construire en 2026 un système Linux qui démarre depuis un environnement temporaire, présente un système distant comme base, ajoute une couche locale d'écriture avec OverlayFS, lance le système final sans second redémarrage, puis migre le stockage local nécessaire vers un support persistant sans interrompre l'espace utilisateur ?

La réponse recherchée ne sera pas un simple « oui » ou « non ».

Le projet produira une **chaîne de preuves expérimentales**, avec :

- les hypothèses ;
- les prérequis ;
- les versions exactes des composants ;
- les étapes de construction ;
- les observations ;
- les mesures ;
- les échecs ;
- les corrections ;
- les résultats reproductibles ;
- les limites de la conclusion.

## Origine du projet

Le point de départ est l'article :

- Étienne Dublé, « Le test de Peter », *GNU/Linux Magazine* n°206, juillet 2017.
- Éditeur : Éditions Diamond.
- Article : <https://connect.ed-diamond.com/GNU-Linux-Magazine/glmf-206/le-test-de-peter>

L'article est utilisé comme **hypothèse historique à auditer**, et non comme autorité technique.

Cette distinction est importante : une primitive Linux documentée en 2017 peut avoir changé de comportement, d'interface ou de contraintes ; inversement, une primitive ancienne peut être restée parfaitement pertinente en 2026.

## Le problème technique posé par l'article

La scène de départ contient plusieurs indices qui doivent être expliqués simultanément :

1. le système semble démarrer depuis un support USB ;
2. le système utilisable apparaît sans second redémarrage apparent ;
3. une opération de migration ou d'installation continue alors que le système est déjà utilisable ;
4. une barre de progression accompagne cette opération ;
5. le support temporaire peut finalement être retiré ;
6. le système continue à fonctionner.

Une installation classique ne constitue pas une explication satisfaisante de cet ensemble.

L'intérêt du raisonnement de l'article est donc de transformer une contradiction apparente en problème d'architecture :

```text
Observation
    |
    v
Contradiction apparente
    |
    v
Hypothèse d'architecture
    |
    v
Primitive Linux
    |
    v
Nouvelle contrainte
    |
    v
Nouvelle primitive
    |
    v
Architecture complète
    |
    v
Expérience
    |
    v
Résultat
```

C'est cette méthode que le projet veut conserver.

## Architecture historique reconstituée

L'article assemble progressivement plusieurs mécanismes.

Une représentation simplifiée est :

```text
                    MACHINE
                       |
                       v
                    kernel
                       |
                       v
                   initramfs
                       |
          +------------+------------+
          |                         |
          v                         v
       réseau                     local
          |                         |
          v                         v
       NFS root              stockage USB
          |                         |
          |                    PV / VG / LV
          |                         |
          +------------+------------+
                       |
                       v
                   OverlayFS
                 lowerdir = NFS
                 upperdir = local
                       |
                       v
                système visible
                       |
                       v
                  PID 1 final
                       |
                       v
                 pvmove / LVM
                       |
                       v
                 stockage local
                   persistant
```

Cette architecture est une **reconstruction expérimentale inspirée de l'article**. Elle ne constitue pas une affirmation selon laquelle la production télévisée utilisait exactement cette architecture.

## Audit technique 2026

L'audit initial montre une différence essentielle entre les primitives individuelles et leur combinaison.

| Élément | Verdict 2026 | Commentaire |
| --- | --- | --- |
| Initramfs | `EXACT` | Le kernel peut extraire un initramfs et exécuter `/init` comme PID 1. |
| Archive cpio | `TOUJOURS VALABLE` | Le format reste la base du mécanisme initramfs, avec compression éventuelle. |
| NFS root | `TOUJOURS VALABLE` | Linux sait toujours démarrer avec un root filesystem NFS. |
| `exec` | `EXACT` | Remplace l'image du processus courant et permet de conserver le PID. |
| `chroot` | `APPROXIMATIF` dans le contexte de l'article | Change la racine de résolution des chemins, mais n'est pas à lui seul une transition complète de root filesystem. |
| `switch_root` / `pivot_root` | `POINT CRITIQUE` | Leur rôle doit être étudié explicitement dans une implémentation moderne. |
| PID 1 | `EXACT` | `/init` de l'initramfs est lancé comme PID 1 dans l'espace de PID initial. |
| OverlayFS | `TOUJOURS VALABLE` | Le modèle lowerdir/upperdir/workdir reste fondamentalement pertinent. |
| NFS comme lowerdir | `PLAUSIBLE` | Compatible avec le modèle visé, mais doit être testé dans la combinaison exacte retenue. |
| NFS comme upperdir | `À ÉCARTER` | Les contraintes de l'upper filesystem rendent cette configuration inadaptée. |
| Écritures dans upperdir | `EXACT` | Les modifications doivent être persistées dans la couche supérieure. |
| LVM | `TOUJOURS VALABLE` | Toujours utilisé comme couche de gestion du stockage. |
| `pvmove` | `EXACT` | Déplace les physical extents entre PV, y compris pendant l'utilisation des volumes selon les contraintes LVM. |
| Progression de `pvmove` | `EXACT` | Le mécanisme fournit une progression exploitable pour une observation expérimentale. |
| `debootstick` | `TOUJOURS VALABLE` | Le projet existe toujours dans l'écosystème Debian/Ubuntu. |
| Architecture complète | `NON DÉMONTRÉE` | La compatibilité de toutes les briques doit être vérifiée de bout en bout. |

## Ce qui est réellement démontré par l'audit documentaire

Les primitives essentielles sont réelles et toujours disponibles :

```text
initramfs
    |
    +--> NFS
    |
    +--> OverlayFS
    |
    +--> PID 1 / exec
    |
    +--> LVM
           |
           +--> pvmove
```

Cela permet de conclure :

> **L'idée générale reste techniquement plausible en 2026.**

Mais cela ne permet pas encore de conclure :

> **La scène est reproductible telle quelle.**

Cette différence constitue précisément le problème expérimental du dépôt.

## Le point méthodologique central

Une erreur fréquente dans ce type d'analyse consiste à effectuer le raisonnement suivant :

```text
Primitive A fonctionne.
Primitive B fonctionne.
Primitive C fonctionne.
--------------------------------
Donc A + B + C fonctionne.
```

Ce raisonnement est invalide sans test d'intégration.

Le projet adopte donc la chaîne suivante :

```text
Documentation
     |
     v
Hypothèse
     |
     v
Test isolé
     |
     v
Test d'intégration
     |
     v
Test de bout en bout
     |
     v
Conclusion bornée
```

Une documentation de primitive n'est pas une preuve de compatibilité de la combinaison.

## Hypothèses expérimentales

### H1 — Initramfs

Le kernel peut charger un initramfs contenant un `/init` exécuté comme PID 1 et suffisamment d'outils pour préparer l'environnement de démarrage.

**Statut initial :** `CONFIRMÉ` conceptuellement, `À REPRODUIRE` expérimentalement.

### H2 — Root distant

Le système peut utiliser un root filesystem fourni par NFS pendant la phase de démarrage.

**Statut initial :** `CONFIRMÉ` conceptuellement, `À REPRODUIRE` dans la configuration retenue.

### H3 — OverlayFS

Un système distant peut être utilisé comme couche inférieure d'un OverlayFS tandis qu'une couche supérieure locale reçoit les écritures.

**Statut initial :** `CONDITIONNEL`.

Le test doit notamment confirmer la compatibilité exacte entre :

- la version du kernel ;
- NFS ;
- le filesystem local ;
- les options OverlayFS ;
- le comportement des opérations de copy-up.

### H4 — PID 1

Le processus `/init` de l'initramfs peut préparer l'environnement puis remplacer son image par l'init du système final avec `exec`, en conservant son PID.

**Statut initial :** `CONFIRMÉ` au niveau du mécanisme.

### H5 — Transition vers le système final

La transition initramfs vers le système final doit être réalisée sans laisser derrière elle un environnement temporaire qui empêche la continuité du système.

**Statut initial :** `À TESTER`.

Le projet comparera explicitement :

- `chroot` ;
- `switch_root` ;
- `pivot_root`.

Ils ne seront pas considérés comme interchangeables.

### H6 — Migration LVM

Le stockage local utilisé par le système peut être intégré dans LVM puis migré vers un autre PV avec `pvmove`.

**Statut initial :** `CONFIRMÉ` au niveau de LVM, `À INTÉGRER` dans l'architecture complète.

### H7 — Continuité du système

La migration du stockage peut se poursuivre pendant que les processus utilisateurs continuent leur activité.

**Statut initial :** `À DÉMONTRER`.

### H8 — Survie après retrait

Une fois les données nécessaires transférées, l'environnement de démarrage temporaire peut être retiré sans interrompre le système utilisateur.

**Statut initial :** `À DÉMONTRER`.

## Le rôle particulier de `exec`

L'article identifie correctement un mécanisme important.

Si `/init` est PID 1 :

```text
PID 1
  |
  +-- /init
```

et que celui-ci exécute :

```text
exec <nouvel-init>
```

le processus n'est pas créé comme enfant.

L'image du processus courant est remplacée :

```text
Avant :

PID 1
  |
  +-- /init

Après exec :

PID 1
  |
  +-- /sbin/init
```

Cette propriété est importante pour la construction proposée par l'article.

Cependant, `exec` ne résout pas à lui seul toute la transition de l'initramfs vers le système final.

## `chroot` n'est pas `switch_root`

C'est l'un des points sur lesquels l'article doit être modernisé.

`chroot` modifie la racine utilisée par le processus pour la résolution des chemins. Il ne constitue pas, à lui seul, une opération complète de remplacement du root mount.

Il faut donc distinguer :

```text
chroot
    |
    +--> change la racine de résolution du processus
```

de :

```text
switch_root / pivot_root
    |
    +--> réorganise le root filesystem et les mounts
```

Une reproduction 2026 devra déterminer quelle transition est effectivement nécessaire et pourquoi.

## Le rôle d'OverlayFS

OverlayFS permet de présenter plusieurs arbres de fichiers sous une vue unique.

Le modèle minimal est :

```text
lowerdir
    |
    | fichiers de base
    v
OverlayFS
    ^
    | écritures / copy-up
    |
upperdir
```

Dans notre scénario :

```text
NFS
 |
 +-- lowerdir
 |
 v
OverlayFS
 ^
 |
 +-- upperdir local
```

L'intérêt est de pouvoir conserver un système de base distant tout en donnant au système actif une couche d'écriture locale.

## Le problème critique de l'upperdir

Cette construction produit immédiatement une contrainte.

Si :

```text
upperdir = USB
```

alors les nouvelles écritures peuvent continuer à être placées sur l'USB.

Donc :

```text
Utilisateur
    |
    v
OverlayFS
    |
    v
upperdir
    |
    v
USB
```

Retirer l'USB trop tôt rend le système incohérent.

Cette observation est fondamentale : elle montre pourquoi l'utilisation d'OverlayFS ne suffit pas à expliquer la scène.

Il faut encore expliquer comment la couche locale devient persistante.

## Le rôle de LVM

LVM change la nature du problème.

Il ne s'agit plus nécessairement de copier des fichiers :

```text
cp -a source destination
```

mais de déplacer les extents physiques qui constituent les volumes logiques :

```text
PV source
   |
   v
VG
   |
   v
LV
   |
   v
filesystem
```

Puis :

```text
PV USB
   |
   | pvmove
   v
PV disque
```

Le filesystem continue à voir le même LV.

C'est précisément cette abstraction qui rend une migration en cours d'utilisation envisageable.

## Le rôle de `pvmove`

`pvmove` déplace les physical extents d'un PV vers un ou plusieurs PV de destination.

Le modèle conceptuel est :

```text
Avant :

PV USB
+--------+--------+--------+
| PE 001 | PE 002 | PE 003 |
+--------+--------+--------+

Après migration :

PV disque
+--------+--------+--------+
| PE 001 | PE 002 | PE 003 |
+--------+--------+--------+
```

Le LV et le filesystem situés au-dessus ne sont pas redéfinis comme lors d'une copie de fichiers classique.

La progression de `pvmove` constitue également une piste crédible pour expliquer la barre de progression observée dans la scène.

## Pourquoi `pvmove` ne constitue pas à lui seul une preuve

Il faut éviter une seconde simplification :

> « `pvmove` fonctionne à chaud, donc notre système peut nécessairement migrer à chaud. »

La vraie question est :

```text
OverlayFS
    |
    v
upperdir
    |
    v
filesystem
    |
    v
LV
    |
    v
PV USB
    |
    | pvmove
    v
PV disque
```

**Cette chaîne complète doit fonctionner dans l'environnement choisi.**

C'est l'une des expériences centrales du projet.

## Le cas `debootstick`

`debootstick` est particulièrement intéressant parce qu'il fournit un précédent concret.

L'article l'utilise pour montrer qu'une transformation d'environnement Debian en système bootable peut être automatisée et qu'une stratégie de migration LVM peut être mise en œuvre.

Le projet actuel ne prendra cependant pas le code de `debootstick` comme preuve automatique de notre architecture.

Il servira plutôt de :

- référence historique ;
- source d'inspiration ;
- implémentation à étudier ;
- point de comparaison ;
- objet d'audit de compatibilité 2026.

Le fait que `debootstick` existe encore dans les distributions Debian/Ubuntu contemporaines est lui-même un résultat intéressant de l'audit documentaire.

## Ce que le projet doit encore prouver

Les maillons les plus importants restent ceux qui combinent plusieurs primitives.

### 1. NFS + OverlayFS

Il faut vérifier la combinaison exacte :

```text
NFS lowerdir
      +
local upperdir
      |
      v
OverlayFS
```

### 2. OverlayFS + LVM

Il faut vérifier que l'upperdir peut être placé sur le stockage LVM utilisé pour l'expérience et que ce stockage peut être migré sans casser la vue OverlayFS.

### 3. Migration pendant l'espace utilisateur

Il faut vérifier que :

```text
pvmove
   |
   +---- système actif
   |
   +---- processus actifs
   |
   +---- écritures simultanées
```

reste cohérent.

### 4. Transition initramfs → système final

Il faut vérifier ce qui arrive au processus de migration lorsque l'initramfs cesse d'être le root filesystem actif.

### 5. Retrait du support temporaire

Enfin :

```text
support temporaire
       |
       v
migration terminée
       |
       v
retrait
       |
       v
système opérationnel
```

doit être démontré et observé.

## Architecture expérimentale 2026

La première cible sera une machine virtuelle.

```text
+------------------------------------------------------+
|                    VM de test                        |
|                                                      |
|  kernel                                              |
|     |                                                |
|     v                                                |
|  initramfs                                            |
|     |                                                |
|     +---- réseau ----> serveur NFS                   |
|     |                                                |
|     +---- disque A --> stockage temporaire           |
|     |                     |                          |
|     |                     +--> PV / VG / LV          |
|     |                                                |
|     v                                                |
|  OverlayFS                                           |
|     |                                                |
|     v                                                |
|  système utilisateur                                |
|     |                                                |
|     v                                                |
|  pvmove : disque A ---> disque B                     |
|                                                      |
+------------------------------------------------------+
```

La virtualisation offre plusieurs avantages :

- snapshots ;
- répétabilité ;
- isolation ;
- récupération après échec ;
- contrôle du matériel virtuel ;
- observation des disques ;
- possibilité de simuler le retrait d'un support.

Le matériel physique pourra être envisagé ensuite si la VM laisse une question qui dépend réellement du matériel.

## Stratégie expérimentale

Chaque mécanisme doit être validé isolément avant intégration.

### Test 1 — Initramfs

Objectif :

```text
kernel
  |
  v
initramfs
  |
  v
/init
  |
  v
PID 1
```

Preuves recherchées :

- `/init` exécuté ;
- PID observé ;
- modules disponibles ;
- outils nécessaires présents.

### Test 2 — NFS root

Objectif :

```text
kernel
  |
  v
initramfs
  |
  v
réseau
  |
  v
NFS root
```

Preuves recherchées :

- montage NFS ;
- root effectivement fourni par NFS ;
- comportement en cas de latence ou de perte réseau ;
- versions et options utilisées.

### Test 3 — OverlayFS

Objectif :

```text
NFS lowerdir
      +
local upperdir
      |
      v
OverlayFS
```

Preuves recherchées :

- lecture des fichiers du lower ;
- création de fichiers ;
- modification de fichiers existants ;
- copy-up ;
- emplacement réel des écritures ;
- comportement après remontage.

### Test 4 — Transition PID 1

Objectif :

```text
/init
  |
  | exec
  v
init système
```

Puis comparaison :

```text
chroot
switch_root
pivot_root
```

Preuves recherchées :

- PID ;
- root mount ;
- mount namespace ;
- processus résiduels ;
- mounts résiduels ;
- accès aux systèmes de fichiers après transition.

### Test 5 — LVM

Objectif :

```text
PV A
 |
 v
VG
 |
 v
LV
 |
 v
filesystem
```

Puis ajout :

```text
PV B
```

et migration :

```text
PV A ---> PV B
```

### Test 6 — Migration en activité

Pendant `pvmove`, le système devra :

- lire des fichiers ;
- écrire des fichiers ;
- maintenir des processus actifs ;
- produire des données observables.

L'objectif est de démontrer la continuité, pas simplement l'absence d'erreur de `pvmove`.

### Test 7 — Retrait

Après migration :

```text
support temporaire
       |
       v
désactivation
       |
       v
retrait logique
       |
       v
système actif
```

Le système doit continuer à fonctionner et les données doivent rester accessibles.

### Test 8 — Bout en bout

Uniquement après réussite des tests précédents :

```text
boot
  |
  v
initramfs
  |
  v
NFS
  |
  v
OverlayFS
  |
  v
init final
  |
  v
pvmove
  |
  v
retrait du support
  |
  v
système opérationnel
```

## Matrice de validation

Chaque expérience sera classée selon une nomenclature explicite.

| Statut | Signification |
| --- | --- |
| `CONFIRMÉ` | Le mécanisme est documenté et le test local confirme son comportement attendu. |
| `PARTIEL` | Une partie de l'hypothèse est confirmée, mais une dépendance reste non vérifiée. |
| `ÉCHEC` | Le scénario ne fonctionne pas dans les conditions expérimentales retenues. |
| `CONDITIONNEL` | Le scénario fonctionne uniquement sous des contraintes explicitement documentées. |
| `NON TESTÉ` | L'expérience n'a pas encore été réalisée. |
| `INVALIDÉ` | Une hypothèse de départ est démontrée incompatible avec l'architecture retenue. |

Aucune conclusion globale ne sera tirée à partir d'une simple réussite individuelle.

## Observabilité

Une expérience crédible doit laisser des traces.

Le projet enregistrera au minimum :

- version du kernel ;
- distribution ;
- version de l'initramfs ;
- outil d'initramfs utilisé ;
- version de LVM2 ;
- version de NFS pertinente ;
- filesystem utilisé ;
- options de montage ;
- topologie des PV, VG et LV ;
- topologie OverlayFS ;
- PID de `/init` avant et après transition ;
- mount namespace ;
- processus actifs ;
- état de `pvmove` ;
- progression de la migration ;
- erreurs kernel ;
- journaux du système ;
- état du stockage avant et après migration ;
- résultat du retrait du support temporaire.

Les expériences devront être accompagnées de commandes et de sorties suffisamment précises pour permettre leur reproduction.

## Critères de réussite

Le **Test de Peter 2026** sera considéré comme réussi uniquement si une expérimentation documentée démontre simultanément les propriétés suivantes :

1. le système démarre depuis l'environnement temporaire ;
2. le système final devient utilisable sans second redémarrage ;
3. le système utilise effectivement la couche distante et la couche locale prévues ;
4. les écritures utilisateur sont dirigées vers le stockage local attendu ;
5. le stockage local peut être migré selon la stratégie LVM retenue ;
6. les processus utilisateurs restent opérationnels pendant la migration ;
7. la transition ne laisse pas de dépendance cachée au support temporaire ;
8. le support temporaire peut être retiré ;
9. le système reste opérationnel après ce retrait ;
10. l'ensemble est reproductible à partir de la documentation du dépôt.

Une réussite partielle sera documentée comme telle.

## Ce que le projet ne prétend pas démontrer

Le projet ne cherchera pas à démontrer :

- que la scène télévisée a réellement été réalisée avec cette architecture ;
- que l'architecture de l'article est la seule explication possible ;
- qu'une preuve de concept en VM est équivalente à une machine physique ;
- qu'un mécanisme Linux moderne se comporte exactement comme en 2017 ;
- qu'une combinaison de primitives compatibles séparément est nécessairement compatible de bout en bout ;
- qu'un résultat positif dans une distribution prouve le comportement dans toutes les distributions ;
- qu'un succès avec une version de kernel permet de généraliser le résultat à toutes les versions.

La conclusion finale devra rester proportionnée aux expériences effectivement réalisées.

## Ce que nous cherchons aussi à mesurer

Le projet ne vise pas uniquement le verdict final.

Il cherchera à identifier **où se situe réellement la difficulté**.

Par exemple :

```text
initramfs
    |
    | facile
    v
NFS root
    |
    | facile / conditionnel
    v
OverlayFS
    |
    | ?
    v
transition PID 1
    |
    | ?
    v
LVM
    |
    | facile
    v
pvmove
    |
    | ?
    v
retrait du support
```

Le résultat le plus intéressant pourrait donc être un échec intermédiaire bien caractérisé.

Un échec reproductible et expliqué est plus utile qu'un « ça marche » non documenté.

## Principes de rigueur

Le projet adopte les règles suivantes :

1. Une correction automatisée n'est pas une validation.
2. Une documentation de primitive n'est pas une preuve de compatibilité de la combinaison.
3. Une preuve de concept n'est pas une reproduction historique.
4. Une réussite dans une VM n'est pas automatiquement une réussite sur matériel physique.
5. Toute hypothèse importante doit être falsifiable.
6. Les échecs sont des résultats expérimentaux et seront conservés.
7. Les versions exactes des composants doivent être enregistrées.
8. Les conclusions doivent rester proportionnées aux observations.
9. Toute modification importante de l'architecture doit être documentée.
10. Une solution qui fonctionne mais viole une hypothèse centrale doit être classée comme telle, et non présentée comme une validation de l'hypothèse initiale.

## Sécurité et isolation

Les expériences concernent des mécanismes bas niveau de :

- démarrage ;
- systèmes de fichiers ;
- réseau ;
- stockage ;
- namespaces ;
- LVM.

Elles doivent être réalisées dans un environnement expérimental isolé.

**Ne pas appliquer les procédures expérimentales à un système de production sans validation préalable et sauvegarde vérifiée.**

La première campagne privilégiera une VM dont les disques et les interfaces réseau peuvent être détruits et recréés.

## Structure prévue du dépôt

```text
test-de-peter-2026/
├── README.md
├── README.fr.md
├── LICENSE
├── docs/
│   ├── architecture.md
│   ├── experimental-method.md
│   ├── validation-matrix.md
│   ├── findings.md
│   └── references.md
├── experiments/
│   ├── 01-initramfs/
│   ├── 02-nfs-root/
│   ├── 03-overlayfs/
│   ├── 04-pid1-transition/
│   ├── 05-lvm-pvmove/
│   ├── 06-live-migration/
│   └── 07-end-to-end/
├── scripts/
├── configs/
└── results/
```

La structure pourra évoluer au fur et à mesure que les expériences révéleront les véritables frontières du système.

## Documentation attendue

Chaque expérience devra idéalement contenir :

```text
experiments/XX-nom/
├── README.fr.md
├── README.md
├── config/
├── scripts/
├── logs/
└── results/
```

Le README de chaque expérience devra préciser :

- l'hypothèse ;
- le matériel virtuel ;
- les versions ;
- les prérequis ;
- la procédure ;
- les observations attendues ;
- les observations réelles ;
- les échecs ;
- les conclusions ;
- les limites.

## Références techniques

Les références seront maintenues dans `docs/references.md`.

Les principales familles de documentation utilisées pour l'audit initial sont :

- documentation du kernel Linux sur initramfs ;
- documentation du kernel Linux sur OverlayFS ;
- documentation du kernel Linux sur NFS root ;
- pages de manuel Linux pour `execve`, `chroot`, `pivot_root` et `pvmove` ;
- documentation et code source de `debootstick` ;
- documentation des outils LVM2.

Les références seront associées aux affirmations techniques qu'elles permettent de vérifier.

## Résultat attendu

Le résultat final devra répondre à quatre questions distinctes.

### Question 1 — Les primitives existent-elles encore ?

Réponse attendue : inventaire documenté des mécanismes effectivement disponibles en 2026.

### Question 2 — Les primitives sont-elles compatibles entre elles ?

Réponse attendue : résultats des tests d'intégration.

### Question 3 — La chaîne complète est-elle reproductible ?

Réponse attendue : procédure de reproduction et preuves.

### Question 4 — Jusqu'où peut-on réellement valider l'article ?

Réponse attendue : conclusion bornée par les résultats expérimentaux.

Le projet pourra donc aboutir à l'un des résultats suivants :

```text
VALIDÉ
```

ou :

```text
VALIDÉ SOUS CONDITIONS
```

ou :

```text
PARTIELLEMENT VALIDÉ
```

ou :

```text
INVALIDÉ SUR UN OU PLUSIEURS MAILLONS
```

Aucun de ces statuts ne sera attribué avant la fin de la campagne expérimentale correspondante.

## État actuel

| Domaine | État |
| --- | --- |
| Analyse de l'article de 2017 | `TERMINÉE` |
| Audit technique initial 2026 | `TERMINÉ` |
| Architecture expérimentale | `EN PRÉPARATION` |
| VM de référence | `À CONSTRUIRE` |
| Tests initramfs | `À FAIRE` |
| Test NFS root | `À FAIRE` |
| Test OverlayFS | `À FAIRE` |
| Test transition PID 1 | `À FAIRE` |
| Test LVM / `pvmove` | `À FAIRE` |
| Test migration en activité | `À FAIRE` |
| Test retrait du support | `À FAIRE` |
| Test de bout en bout | `À FAIRE` |
| Rapport de résultats | `À FAIRE` |

## Pourquoi ce projet ?

Le « test de Peter » possède une qualité rare : il transforme une scène apparemment impossible en problème d'ingénierie.

Il oblige à raisonner par couches et à accepter qu'une première solution puisse révéler une nouvelle contrainte :

```text
Observation
    |
    v
Anomalie
    |
    v
Hypothèse
    |
    v
Primitive
    |
    v
Nouvelle contrainte
    |
    v
Nouvelle primitive
    |
    v
Architecture
    |
    v
Expérience
    |
    v
Résultat
```

C'est une méthode qui dépasse largement le cas Linux.

Elle est pertinente pour :

- le reverse engineering ;
- le dépannage système ;
- l'analyse d'architecture ;
- le threat modeling ;
- l'audit technique ;
- la validation de faisabilité ;
- l'analyse critique de scénarios techniques.

Le projet veut pousser cette démarche jusqu'au bout.

Pas de « ça devrait marcher ».

Pas de « Linux peut sûrement le faire ».

**On construit. On mesure. On casse. On corrige. On recommence.**

## Licence

La licence du dépôt sera définie avant la première version publique.
