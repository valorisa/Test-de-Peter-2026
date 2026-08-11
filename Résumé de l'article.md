# Résumé de l'article « Le test de Peter » – GNU/Linux Magazine n°206 (juillet 2017)

Étienne Dublé utilise une conversation fictive avec son ami **Peter (un « testeur compulsif »)** pour décortiquer la faisabilité technique d’un piratage informatique vu dans une série télévisée. L’article mêle humour et pédagogie afin d’expliquer des mécanismes avancés de GNU/Linux.

## 🔎 Contexte
Dans l’épisode analysé, un agent (McGee) pirate l’ordinateur d’un collègue (Devon) à l’aide d’une clé USB bootable. La clé lance une réinstallation discrète de l’OS, puis McGee la débranche à chaud, laissant le système modifié fonctionner sans redémarrage.

## 🧠 Approche pédagogique
Peter décortique la scène étape par étape et propose une explication technique cohérente, en s’appuyant sur des concepts réels de GNU/Linux.

## 🔧 Concepts clés abordés

### 1. **Initramfs et démarrage de l’OS**
- Le noyau Linux charge une archive **initramfs** en RAM, qui contient un mini-système.
- Ce mini-système monte le système de fichiers final (ex. partage **NFS**) et exécute le **vrai système d’init** (`/sbin/init`).
- La commande `exec chroot . /sbin/init` est utilisée pour :
  - Changer la racine du système de fichiers (`chroot`).
  - Conserver le **PID 1** sur le processus d’init (`exec`), indispensable pour la gestion des signaux et des arrêts.

### 2. **Union de fichiers (overlayfs)**
- Pour injecter des modifications sans altérer le système original, Peter propose d’utiliser un montage **overlay** combinant :
  - La **couche inférieure** : l’OS distant (lecture seule, via NFS).
  - La **couche supérieure** : un répertoire `hacks` situé sur la clé USB (lecture/écriture).
- Les modifications (programmes piratés, fichiers de configuration) sont stockées dans la couche supérieure, invisibles pour l’utilisateur.

### 3. **Migration à chaud avec LVM**
- Pour rendre le piratage persistant (après un reboot sans la clé), le système doit **copier les données de la clé vers le disque interne**.
- L’article explique comment **LVM (Logical Volume Management)** permet de :
  - Ajouter un nouveau volume physique (le disque interne).
  - Migrer les données d’un volume physique à un autre **à chaud**, sans arrêter le système.
  - Cette technique est illustrée par une extraction du code de l’outil **debootstick**, qui réalise exactement cette opération.

## ✅ Conclusion
L’article valide la **plausibilité technique** du scénario de la série :
1. Un initramfs modifié monte un overlay combinant NFS et des hacks locaux.
2. Le système d’init est lancé sur cette union, intégrant les modifications.
3. En arrière-plan, LVM migre les données de la clé vers le disque interne, assurant la persistance après retrait de la clé.

## 💡 Portée
L’auteur invite le lecteur à adopter un regard critique et curieux face aux scènes informatiques dans les œuvres de fiction, tout en transmettant des connaissances techniques solides sur le boot, les filesystems et la virtualisation de stockage.
