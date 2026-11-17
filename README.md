# Termux-SSH-Localhost

Guide de diagnostic et de résolution pour se connecter en SSH à **`localhost`** depuis Termux (Android), avec une authentification par clé, sans erreur `Connection refused` et sans avertissements `setlocale`.

Ce dépôt documente une session de dépannage réelle, étape par étape : les symptômes observés, les hypothèses testées, les erreurs de raisonnement rencontrées en chemin, les corrections qui ont fonctionné, et la manière de vérifier que chacune a bien eu l'effet attendu.

---

## Table des matières

1. [Résumé rapide](#1-résumé-rapide)
2. [Contexte et symptômes](#2-contexte-et-symptômes)
3. [Environnement](#3-environnement)
4. [Problème n° 1 : `Connection refused`](#4-problème-n-1--connection-refused)
5. [Problème n° 2 : la clé n'est pas utilisée](#5-problème-n-2--la-clé-nest-pas-utilisée)
6. [Problème n° 3 : avertissements `setlocale`](#6-problème-n-3--avertissements-setlocale)
7. [Confort d'utilisation](#7-confort-dutilisation)
8. [Entretien et sécurité](#8-entretien-et-sécurité)
9. [Tableau de dépannage](#9-tableau-de-dépannage)
10. [Commandes de référence](#10-commandes-de-référence)
11. [Pièges rencontrés](#11-pièges-rencontrés)
12. [Structure du dépôt](#12-structure-du-dépôt)
13. [Contribuer](#13-contribuer)
14. [Licence](#14-licence)

---

## 1. Résumé rapide

Si tu cherches seulement la solution, voici les trois points à retenir :

1. **Sous Termux, `sshd` écoute sur le port 8022, pas 22.** Le port 22 est un port privilégié (inférieur à 1024) que Termux, sans droits root, ne peut pas ouvrir. Il faut donc toujours préciser `-p 8022`.
2. **Pour une connexion par clé, la clé publique doit être dans `~/.ssh/authorized_keys`** du compte sur lequel on se connecte. Si ce n'est pas le cas, le serveur refuse les clés et retombe silencieusement sur le mot de passe.
3. **Les avertissements `setlocale` viennent d'un locale inexistant** (`fr_FR.UTF-8`) exporté via `LC_ALL`. Il faut corriger la source de l'export, puis **redémarrer `sshd`**, qui conserve l'environnement dans lequel il a été lancé.

Séquence minimale :

```sh
pkg install openssh
sshd                                   # lance le serveur (écoute sur 8022)
sshd -T | grep -i '^port'              # confirme le port réel
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519
ssh -p 8022 localhost
```

---

## 2. Contexte et symptômes

L'objectif de départ était simple : ouvrir une session SSH sur le Termux local lui-même (`ssh ... localhost`), en utilisant une clé `ed25519`. C'est utile, par exemple, pour tester une configuration `sshd`, pour valider des clés avant de les déployer vers d'autres machines, ou pour disposer d'un second terminal indépendant.

Les premières tentatives ont toutes échoué avec le même message :

```text
~/.ssh $ ssh -i id_ed25519 localhost
ssh: connect to host localhost port 22: Connection refused
~/.ssh $ ssh -i ./id_ed25519 localhost
ssh: connect to host localhost port 22: Connection refused
~/.ssh $ ssh -i ./id_ed25519.pub localhost -p 8822
ssh: connect to host localhost port 8822: Connection refused
~/.ssh $ ssh -i ./id_ed25519 localhost -p 8822
ssh: connect to host localhost port 8822: Connection refused
```

Trois erreurs distinctes étaient mêlées :

- un **mauvais port** (22, puis 8822 au lieu de 8022) ;
- une **mauvaise option `-i`** dans l'une des commandes (pointant vers la clé publique `.pub` au lieu de la clé privée) ;
- plus tard, une **clé non autorisée** côté serveur, masquée par un repli automatique sur le mot de passe.

S'y est ajouté, une fois la connexion établie, un flot d'avertissements `bash: warning: setlocale: ...` à chaque ouverture de session SSH.

---

## 3. Environnement

| Élément | Valeur |
|---|---|
| Plateforme | Android, application Termux |
| Client et serveur | OpenSSH 10.6p2 (même appareil) |
| Type de clé | `ed25519`, protégée par passphrase |
| Cible | `localhost` (`127.0.0.1`) |
| Port du serveur | `8022` |
| Nom d'utilisateur Termux | de la forme `u0_aXXX` (voir `whoami`) |

Le nom d'utilisateur Termux est attribué par Android et ne se choisit pas. Quand on se connecte à `localhost` sans préciser `utilisateur@`, `ssh` réutilise le nom de l'utilisateur courant, ce qui convient parfaitement ici.

---

## 4. Problème n° 1 : `Connection refused`

### 4.1 Ce que le message signifie

`Connection refused` signifie qu'**aucun processus n'écoute** sur l'adresse et le port visés. Ce n'est **pas** une erreur d'authentification : on n'est même pas arrivé jusqu'à l'échange de clés. Changer de clé, de fichier d'identité ou d'options `-i` ne peut donc rien corriger à ce stade.

Il y a deux causes possibles :

1. le serveur `sshd` n'est pas lancé ;
2. il est lancé, mais sur un autre port que celui qu'on interroge.

### 4.2 Pourquoi pas le port 22 ?

Sur Android, une application ordinaire ne peut pas ouvrir de port inférieur à 1024 sans privilèges root. Termux contourne cela en configurant `sshd` sur le **port 8022** par défaut. Tout client doit donc l'indiquer explicitement :

```sh
ssh -p 8022 localhost
```

Le port `8822`, tenté lors du dépannage, était une simple faute de frappe (inversion de chiffres) : rien n'écoutait dessus.

### 4.3 Vérifier que `sshd` tourne

```sh
pgrep -a sshd
```

Sortie observée :

```text
4666 sshd -D -e
```

Les options `-D` (ne pas passer en arrière-plan) et `-e` (journaliser sur l'erreur standard) indiquent un lancement par un script, un service ou un outil de démarrage tel que Termux:Boot. Elles ont une conséquence pratique : si tu relances `sshd` pour appliquer un changement, fais-le **par le même moyen**, sinon la nouvelle instance ne reproduira pas le même mode de lancement.

Relancer `sshd` à la main pendant qu'une instance tourne déjà ne produit **aucun effet visible** : le second processus échoue en silence parce que le port est déjà occupé. C'est ce qui s'est produit ici (`sshd` sans message, puis `pgrep` montrant toujours le même PID).

### 4.4 Confirmer le port réellement configuré

```sh
sshd -T | grep -i '^port'
```

Sortie observée :

```text
Port 8022
```

L'option `-T` affiche la configuration effective, après lecture de `sshd_config` et de ses inclusions. C'est plus fiable que de lire le fichier à la main.

### 4.5 Résultat

Avec le bon port, la connexion s'établit :

```sh
ssh -p 8022 -i ~/.ssh/id_ed25519 localhost
```

---

## 5. Problème n° 2 : la clé n'est pas utilisée

### 5.1 Le symptôme trompeur

La connexion a réussi, mais **avec un mot de passe** :

```text
u0_a396@localhost's password:
```

L'intention était une authentification par clé. Le comportement de `ssh` est le suivant : il présente chaque clé disponible, et si le serveur les refuse toutes, il propose les méthodes suivantes (`keyboard-interactive`, puis `password`). La connexion « marche », ce qui masque le fait que la clé ne sert à rien.

### 5.2 Lire la sortie de `ssh -v`

Le mode verbeux montre exactement ce qui se passe :

```sh
ssh -v -p 8022 localhost
```

Extraits significatifs :

```text
debug1: Offering public key: .../.ssh/id_rsa RSA SHA256:...
debug1: Authentications that can continue: publickey,password,keyboard-interactive
debug1: Offering public key: .../.ssh/id_ed25519 ED25519 SHA256:...
debug1: Authentications that can continue: publickey,password,keyboard-interactive
debug1: Next authentication method: keyboard-interactive
debug1: Next authentication method: password
Authenticated to localhost ([127.0.0.1]:8022) using "password".
```

Lecture :

- `Offering public key` : le client propose la clé ;
- la ligne `Authentications that can continue` qui suit indique que le serveur **n'a pas accepté** cette clé ;
- à la fin, `using "password"` confirme que c'est le mot de passe qui a ouvert la session.

Si la clé avait été acceptée, on aurait vu `Authenticated to ... using "publickey"`.

### 5.3 Cause

Le serveur n'accepte une clé que si la **clé publique correspondante** figure dans le fichier `~/.ssh/authorized_keys` du compte cible. Posséder la paire de clés côté client ne suffit pas.

### 5.4 Correction

```sh
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519
```

Les permissions comptent : avec `StrictModes` activé (valeur par défaut), `sshd` ignore un `authorized_keys` ou un répertoire `~/.ssh` trop permissifs.

### 5.5 Vérification

Après correction, `ssh` demande la **passphrase de la clé** (et non plus le mot de passe du compte) :

```text
Enter passphrase for key '/data/data/com.termux/files/home/.ssh/id_ed25519':
```

C'est le signe que la clé est acceptée. La passphrase protège la clé privée localement ; elle est distincte du mot de passe du compte.

### 5.6 Règles à retenir sur `-i`

- `-i` désigne un **fichier d'identité**, c'est-à-dire la clé **privée** (`id_ed25519`). Ne jamais lui donner le fichier `.pub`.
- `-i` n'est pas obligatoire : `ssh` essaie par défaut `id_rsa`, `id_ecdsa`, `id_ed25519`, etc. dans `~/.ssh`.
- Une clé n'a de sens, en connexion loopback, que si sa partie publique est dans `authorized_keys` **du même compte**.

### 5.7 Cas d'une paire destinée à une autre machine

Une paire telle que `id_ed25519_CPH1917_to_Asus_ROG_Z13` illustre une règle générale : dans un scénario client → serveur, c'est la clé **publique du client** qui doit figurer dans `authorized_keys` **du serveur**. Ajouter la clé publique sur la machine cliente ne sert à rien.

---

## 6. Problème n° 3 : avertissements `setlocale`

### 6.1 Le symptôme

Chaque ouverture de session SSH affichait un bloc de lignes de ce type :

```text
bash: warning: setlocale: LC_CTYPE: cannot change locale (fr_FR.UTF-8): No such file or directory
bash: warning: setlocale: LC_COLLATE: cannot change locale (fr_FR.UTF-8): No such file or directory
bash: warning: setlocale: LC_ALL: cannot change locale (fr_FR.UTF-8): No such file or directory
```

Le locale `fr_FR.UTF-8` n'est pas installé dans cet environnement Termux. Bash tente de l'appliquer, échoue, et le signale.

### 6.2 Première piste, fausse

La première hypothèse était un `export` de `fr_FR` dans `~/.bashrc` ou `~/.profile`. La vérification a **infirmé** cette hypothèse :

```text
~ $ cat ~/.bashrc
export LANG=en_US.UTF-8
...
~ $ cat ~/.profile
#export LANG=en_US.UTF-8
#export LC_ALL=en_US.UTF-8
```

Ni l'un ni l'autre ne contenait `fr_FR`. Il fallait chercher plus largement.

### 6.3 Localiser la source

```sh
env | grep -E '^(LANG|LC_)'
grep -rn "fr_FR" ~/.bash_profile ~/.bash_login ~/.bashrc ~/.profile ~/.termux \
  ~/.ssh/environment $PREFIX/etc/profile $PREFIX/etc/profile.d \
  $PREFIX/etc/bash.bashrc $PREFIX/etc/ssh/sshd_config 2>/dev/null
```

Résultat :

```text
LANG=en_US.UTF-8
LC_ALL=fr_FR.UTF-8
.../usr/etc/profile.d/locale.sh:1:export LANG=fr_FR.UTF-8
.../usr/etc/profile.d/locale.sh:2:export LC_ALL=fr_FR.UTF-8
```

La source est `$PREFIX/etc/profile.d/locale.sh`. Ce fichier n'a pas été identifié comme appartenant à une installation Termux standard : il provient vraisemblablement d'une configuration locale ou d'un script de mise en place.

### 6.4 Pourquoi `export LANG=en_US.UTF-8` dans `~/.bashrc` ne servait à rien

Les variables `LC_*` ont priorité sur `LANG`, et **`LC_ALL` l'emporte sur toutes les autres**. Tant que `LC_ALL=fr_FR.UTF-8` est défini, la valeur de `LANG` est ignorée pour tous les aspects du locale. Corriger `LANG` seul ne pouvait donc pas résoudre le problème.

### 6.5 Pourquoi seulement en SSH

Le shell local de Termux et une session SSH ne partagent pas la même chaîne d'initialisation. Le processus `sshd` conserve l'environnement dans lequel il a été lancé et le transmet aux sessions qu'il ouvre. On peut l'inspecter directement :

```sh
tr '\0' '\n' < /proc/$(pgrep -x sshd | head -n1)/environ | grep -E '^(LANG|LC_)'
```

Sortie observée avant correction :

```text
LANG=fr_FR.UTF-8
LC_ALL=fr_FR.UTF-8
```

### 6.6 Choisir le locale de remplacement

Tester qu'un locale ne produit aucun avertissement, **sans** modifier les variables du shell courant :

```sh
env LC_ALL=en_US.UTF-8 bash -c true
env LC_ALL=C.UTF-8 bash -c true
```

Une sortie vide signifie que le locale est utilisable. Ici, `en_US.UTF-8` était silencieux. `C.UTF-8` constitue le repli si ce n'était pas le cas.

### 6.7 Correction

```sh
sed -i 's/fr_FR\.UTF-8/en_US.UTF-8/' $PREFIX/etc/profile.d/locale.sh
cat $PREFIX/etc/profile.d/locale.sh
```

Contenu obtenu :

```sh
export LANG=en_US.UTF-8
export LC_ALL=en_US.UTF-8
```

Il faut ensuite :

1. **fermer et rouvrir Termux** pour que le shell local recharge `profile.d` ;
2. **redémarrer `sshd` depuis une session locale** (jamais depuis une session SSH, qui serait coupée) :

   ```sh
   pkill sshd; sshd
   ```

   Si `sshd` est lancé par un script ou par Termux:Boot, relance-le par ce moyen.
3. **vérifier l'environnement du processus** :

   ```sh
   tr '\0' '\n' < /proc/$(pgrep -x sshd | head -n1)/environ | grep -E '^(LANG|LC_)'
   ```

   Sortie attendue :

   ```text
   LANG=en_US.UTF-8
   LC_ALL=en_US.UTF-8
   ```

### 6.8 Constat

Après ces trois étapes, une nouvelle session SSH s'ouvre sans aucun avertissement. Noter que fermer Termux ne suffisait pas : au premier contrôle, `sshd` tournait toujours avec `fr_FR.UTF-8`. Seul le redémarrage explicite de `sshd` a appliqué le nouvel environnement.

### 6.9 Ce qu'il ne faut pas faire

Ne crée pas un fichier `~/.bash_profile` pour contourner le problème. Lorsque `~/.bash_profile` existe, bash cesse de lire `~/.profile` en shell de connexion, et tout ce qui y est défini (par exemple `JAVA_HOME`) disparaît.

---

## 7. Confort d'utilisation

### 7.1 Ne plus taper le port : alias dans `~/.ssh/config`

```text
Host termux-local
    HostName localhost
    Port 8022
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

Usage :

```sh
ssh termux-local
```

`IdentitiesOnly yes` limite les tentatives à la clé indiquée. Sans cela, `ssh` propose d'abord toutes les clés qu'il trouve, ce qui peut épuiser le nombre d'essais autorisés par le serveur.

### 7.2 Ne plus retaper la passphrase : `ssh-agent`

```sh
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

L'agent vit tant que la session Termux vit. Il faut le relancer à chaque démarrage de Termux, ou utiliser un outil comme `keychain` pour automatiser cette étape.

---

## 8. Entretien et sécurité

### 8.1 Doublons dans `authorized_keys`

Relancer `cat ... >> authorized_keys` ajoute la clé une nouvelle fois. Ce n'est pas bloquant, mais on peut nettoyer :

```sh
sort -u -o ~/.ssh/authorized_keys ~/.ssh/authorized_keys
```

Pour éviter le doublon dès l'ajout :

```sh
grep -qF "$(cat ~/.ssh/id_ed25519.pub)" ~/.ssh/authorized_keys \
  || cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
```

Pour inspecter les empreintes des clés autorisées :

```sh
ssh-keygen -lf ~/.ssh/authorized_keys
```

### 8.2 Désactiver le mot de passe, une fois la clé validée

Une fois la connexion par clé vérifiée (`using "publickey"` dans `ssh -v`), on peut durcir le serveur en ajoutant dans `$PREFIX/etc/ssh/sshd_config` :

```text
PasswordAuthentication no
```

Puis redémarrer `sshd`. **Ne le fais pas avant d'avoir confirmé que la clé fonctionne**, sous peine de te verrouiller hors de ta propre session SSH. Garde une session locale ouverte pendant le test.

### 8.3 Limiter l'écoute au loopback

Si le serveur ne doit servir que des connexions locales, on peut restreindre l'écoute :

```text
ListenAddress 127.0.0.1
```

Par défaut, `sshd` écoute sur toutes les interfaces, donc aussi sur le réseau local (Wi-Fi). Selon l'usage, c'est soit voulu, soit à restreindre.

### 8.4 Protection de la clé privée

- La clé privée doit rester en `chmod 600`.
- La passphrase est une protection supplémentaire utile : conserve-la.
- Ne jamais versionner de clé privée dans un dépôt Git. Ce dépôt ne contient aucune clé, aucun nom d'utilisateur réel et aucune empreinte réelle.

---

## 9. Tableau de dépannage

| Symptôme | Cause probable | Vérification | Correction |
|---|---|---|---|
| `Connection refused` sur le port 22 | Mauvais port | `sshd -T \| grep -i '^port'` | `ssh -p 8022 localhost` |
| `Connection refused` sur 8022 | `sshd` non lancé | `pgrep -a sshd` | `sshd` |
| `sshd` sans effet visible | Instance déjà active, port occupé | `pgrep -a sshd` | `pkill sshd; sshd` |
| Mot de passe demandé malgré `-i` | Clé absente de `authorized_keys` | `ssh -v` : `using "password"` | Ajouter la clé `.pub`, ajuster les permissions |
| `-i` ne change rien | `-i` pointe vers la `.pub` | Nom du fichier | Utiliser la clé privée |
| Clé ignorée malgré `authorized_keys` | Permissions trop larges | `ls -ld ~/.ssh ~/.ssh/authorized_keys` | `chmod 700` / `chmod 600` |
| `setlocale: ... fr_FR.UTF-8` | `LC_ALL` pointe vers un locale absent | `env \| grep -E '^(LANG\|LC_)'` | Corriger `profile.d/locale.sh` |
| Avertissements persistants en SSH | `sshd` garde l'ancien environnement | `/proc/<pid>/environ` | `pkill sshd; sshd` depuis une session locale |
| Passphrase redemandée à chaque fois | Pas d'agent | `ssh-add -l` | `ssh-agent` + `ssh-add` |

Pour un diagnostic côté serveur, lancer `sshd` au premier plan avec les logs de debug :

```sh
pkill sshd; sshd -d -p 8022
```

puis se reconnecter depuis une seconde session avec `ssh -v -p 8022 localhost`. Les logs du serveur indiquent la raison exacte d'un refus (permissions, `StrictModes`, format de clé).

---

## 10. Commandes de référence

```sh
# Installer et lancer le serveur
pkg install openssh
sshd

# État et configuration
pgrep -a sshd
sshd -T | grep -i '^port'

# Connexion
ssh -p 8022 localhost
ssh -v -p 8022 localhost           # mode verbeux
ssh termux-local                   # avec l'alias de ~/.ssh/config

# Clés
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys ~/.ssh/id_ed25519
ssh-keygen -lf ~/.ssh/authorized_keys

# Agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Locale
env | grep -E '^(LANG|LC_)'
env LC_ALL=en_US.UTF-8 bash -c true
tr '\0' '\n' < /proc/$(pgrep -x sshd | head -n1)/environ | grep -E '^(LANG|LC_)'
```

---

## 11. Pièges rencontrés

Cette section recense les erreurs de raisonnement survenues pendant le dépannage. Elles sont instructives, car elles se reproduisent facilement.

1. **Chercher dans la clé un problème qui est dans le réseau.** `Connection refused` précède toute authentification. Changer de clé n'a aucune utilité tant que le port est faux.
2. **Passer la clé publique à `-i`.** Le client attend la clé privée.
3. **Prendre une connexion réussie pour une connexion par clé.** Le repli silencieux sur le mot de passe masque l'absence de la clé dans `authorized_keys`. Seul `ssh -v` le révèle.
4. **Supposer l'origine d'un export sans la vérifier.** L'hypothèse `~/.bashrc` / `~/.profile` était fausse. Une recherche `grep -rn` sur l'ensemble des emplacements possibles a localisé la vraie source.
5. **Mal tester un locale avec `LC_ALL=x commande`.** Cette syntaxe modifie temporairement la variable, puis la restaure. Le shell parent émet alors un avertissement lors de la restauration vers l'ancienne valeur invalide, ce qui fait croire que le locale testé est en cause. Utiliser `env LC_ALL=x commande` évite ce biais.
6. **Oublier que `sshd` conserve son environnement.** Modifier le fichier de configuration du locale ne suffit pas : il faut redémarrer le démon.
7. **Redémarrer `sshd` depuis une session SSH.** Cela coupe la session en cours. Toujours le faire depuis une session locale.
8. **Relancer `sshd` alors qu'il tourne déjà.** Aucun message d'erreur n'apparaît, mais rien ne change.

---

## 12. Structure du dépôt

```text
Termux-SSH-Localhost/
├── README.md      # Ce document
└── LICENSE        # À ajouter
```

Le dépôt peut s'enrichir ultérieurement d'un script de diagnostic (`diagnostic.sh`) reprenant les vérifications des sections 4 à 6, et d'un exemple de fichier `ssh_config`.

---

## 13. Contribuer

Les corrections et ajouts sont bienvenus, notamment :

- d'autres causes de `Connection refused` ou de refus de clé rencontrées sous Termux ;
- des variantes selon la version d'Android, de Termux ou d'OpenSSH ;
- des méthodes fiables de démarrage automatique de `sshd`.

Avant de proposer une modification, merci de vérifier la commande sur un appareil réel et de joindre la sortie observée. Les résultats non vérifiés n'ont pas leur place dans ce guide.

---

## 14. Licence

À définir par l'auteur du dépôt. Ajouter un fichier `LICENSE` à la racine une fois le choix fait.
