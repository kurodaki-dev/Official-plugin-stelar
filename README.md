## documentation




# Stelar 3.0 Ollama — Documentation complète

Assistant IA en terminal, sans dépendance graphique. Fonctionne dans Termux, Linux, macOS. Interface façon "Claude Code" : bannière d'accueil, boîtes encadrées, spinner animé, couleurs personnalisables, système de plugins extensible.

---

## Sommaire

1. [Installation](#1-installation)
2. [Démarrage et premier lancement](#2-démarrage-et-premier-lancement)
3. [Structure des fichiers](#3-structure-des-fichiers)
4. [Utiliser le chat](#4-utiliser-le-chat)
5. [Toutes les commandes](#5-toutes-les-commandes)
   - [/help](#help)
   - [/config](#config)
   - [/settings](#settings)
   - [/theme](#theme)
   - [/input](#input)
   - [/memory](#memory)
   - [/reset](#reset)
   - [/clear](#clear)
   - [/plugin](#plugin)
   - [/exit, /quit](#exit-quit)
6. [Backends IA : Cloud vs Local](#6-backends-ia--cloud-vs-local)
7. [Le système d'actions (write_file, execute...)](#7-le-système-dactions-write_file-execute)
8. [Le fichier config.json en détail](#8-le-fichier-configjson-en-détail)
9. [Construire un plugin](#9-construire-un-plugin)
10. [Référence complète de l'API plugin](#10-référence-complète-de-lapi-plugin)
11. [Dépannage](#11-dépannage)

---

## 1. Installation

Stelar est un script Python unique (`stelar.py`) plus un dossier `plugins/`. Une seule dépendance externe :

```bash
pip install requests
```

Sur Termux, si `pip` seul échoue :

```bash
pkg install python
pip install requests
```

Aucune autre dépendance : pas de Tkinter, pas de serveur, pas de compilation.

---

## 2. Démarrage et premier lancement

```bash
python3 stelar.py
```

Au tout premier lancement, Stelar crée automatiquement un dossier `storage/` à côté du script, avec :

- `config.json` — tes réglages (nom, modèle, couleurs, backend...)
- `memory.json` — l'historique de conversation
- `plugins.json` — la liste des plugins activés
- `.cli_history` — l'historique des lignes tapées (flèches haut/bas)

Tu n'as rien à créer toi-même, tout se génère avec des valeurs par défaut.

Au démarrage, une bannière s'affiche :

```
──────────────────────────────────────────────────
  ✻ Stelar 3.0 Ollama  ·  assistant en terminal
──────────────────────────────────────────────────
  cwd      /chemin/courant
  backend  ollama cloud
  modèle   gemma4:31b-cloud
  clé api  non définie — /config <clé>
  mémoire  0 message(s)
  plugins  0/2 actif(s) — /plugin list
──────────────────────────────────────────────────
  Tapez /help pour les commandes, /exit pour quitter.
```

Elle te dit immédiatement où tu en es : quel backend est actif, si une clé API est nécessaire, combien de plugins sont disponibles.

**Avant de discuter**, il faut soit :
- configurer une clé API cloud avec `/config <ta_clé>`, soit
- basculer sur un serveur Ollama local avec `/settings` (voir [section 6](#6-backends-ia--cloud-vs-local))

---

## 3. Structure des fichiers

```
mon_dossier/
├── stelar.py                        ← le programme
├── storage/                          ← créé automatiquement
│   ├── config.json
│   ├── memory.json
│   ├── plugins.json
│   └── .cli_history
└── plugins/                          ← tes plugins
    ├── couleur/
    │   ├── plugin.json
    │   └── plugin.py
    └── personnalisation/
        ├── plugin.json
        └── plugin.py
```

`storage/` ne doit jamais être éditée à la main pendant que Stelar tourne (les écritures sont atomiques mais concurrentes à l'édition manuelle, ça peut créer des incohérences). Tu peux la sauvegarder ou la copier quand Stelar est fermé.

---

## 4. Utiliser le chat

Tape n'importe quel texte sans `/` devant, et il part vers l'IA :

```
Utilisateur › explique-moi ce qu'est une closure en JS
```

Pendant la génération, un spinner tourne :

```
  ✻ Réflexion… (3s)
```

Dès que le premier morceau de réponse arrive, le spinner disparaît proprement et le texte s'affiche en streaming (mot par mot, comme du texte qui s'écrit en direct) sous un label :

```
✻ Stelar
Une closure est une fonction qui...
```

Chaque échange (ton message + la réponse) est automatiquement sauvegardé dans `memory.json` et réinjecté dans les 20 prochains messages envoyés à l'IA, pour qu'elle garde le contexte de la conversation.

**Interrompre une génération en cours** : `Ctrl+C`. Ça arrête proprement le flux réseau sans planter le programme.

---

## 5. Toutes les commandes

### `/help`

Affiche la liste des commandes intégrées, plus — s'il y en a — la liste des commandes ajoutées par tes plugins actifs, avec le nom du plugin entre parenthèses.

---

### `/config`

Raccourci rapide pour définir la clé API et le modèle du backend **cloud** uniquement.

```
/config <clé_api> [modèle]
```

Exemples :
```
/config sk-abcd1234
/config sk-abcd1234 gemma4:31b-cloud
```

Si tu veux changer de backend (passer en local) ou régler autre chose, utilise `/settings`.

---

### `/settings`

L'assistant de configuration complet, en 3 étapes séquentielles. Répond aux questions ou laisse vide pour garder la valeur actuelle (affichée entre crochets).

**Étape 1 — identité**
- Ton nom (affiché dans le prompt et utilisé par l'IA pour te nommer)
- Nom de l'IA (affiché devant ses réponses, ex: `✻ Stelar`)

**Étape 2 — backend IA**
Tu choisis `1` (cloud) ou `2` (local) :

- **Si 1 (cloud)** : demande la clé API et le nom du modèle
- **Si 2 (local)** : demande l'adresse du serveur (`localhost` par défaut), **le port** (`11434` par défaut, modifiable), et le nom du modèle local (ex: `llama3.1`, `mistral`)

**Étape 3 — prompt système**
Tu peux réécrire entièrement la "personnalité" de l'IA (les instructions qu'elle reçoit avant chaque conversation). Tape plusieurs lignes, termine par une ligne vide. Si tu laisses vide directement, le prompt système actuel est conservé.

---

### `/theme`

Personnalise les couleurs de l'interface avec des codes CSS.

```
/theme list                    → affiche toutes les cibles et leur couleur actuelle
/theme <cible> <couleur>       → change une couleur
/theme reset                   → restaure toutes les couleurs par défaut
```

**Cibles disponibles :**

| Cible     | Ce qu'elle colore                          |
|-----------|---------------------------------------------|
| `accent`  | Le nom de l'IA, le titre de la bannière, le spinner |
| `user`    | Ton nom dans le prompt de saisie             |
| `success` | Les messages de succès (✔)                   |
| `error`   | Les messages d'erreur (✖)                    |
| `warn`    | Les avertissements (!)                       |
| `box`     | Les bordures des boîtes encadrées            |
| `system`  | Les messages discrets (·)                    |
| `info`    | Les infos (aide, listes...)                  |

**Formats de couleur acceptés :**

| Format | Exemple |
|---|---|
| Hex complet | `#ff6600` |
| Hex court | `#f60` |
| RGB | `rgb(255, 102, 0)` |
| Nom CSS | `orange`, `turquoise`, `crimson`, `teal`... (liste non exhaustive de noms courants) |

Exemples concrets :
```
/theme accent #ff6600
/theme error rgb(220, 20, 60)
/theme success turquoise
```

Chaque changement est **sauvegardé immédiatement** dans `config.json` et **réappliqué automatiquement** à chaque redémarrage — tu n'as rien à refaire.

---

### `/input`

Change la façon dont tu tapes tes messages.

```
/input                  → affiche le mode actuel et l'aide
/input single           → une ligne = un message, Entrée envoie directement (défaut)
/input multi            → plusieurs lignes possibles, une ligne vide envoie le message
```

Le mode `multi` est utile pour coller du texte long ou du code sans que chaque retour à la ligne envoie prématurément le message.

**Astuce :** même en mode `multi`, si la toute première ligne que tu tapes commence par `/`, elle est exécutée immédiatement comme une commande — tu n'as pas besoin de repasser en mode `single` pour taper `/settings` ou `/theme`.

---

### `/memory`

Gère l'historique de conversation persistant.

```
/memory                 → affiche juste le nombre de messages en mémoire
/memory show [N]        → affiche les N derniers messages (5 par défaut)
/memory clear           → vide toute la mémoire (demande confirmation)
```

Exemple :
```
/memory show 10
```

---

### `/reset`

Raccourci identique à `/memory clear` — vide toute la mémoire après confirmation.

---

### `/clear`

Efface l'écran du terminal (équivalent de la commande `clear`/`cls`). N'affecte pas la mémoire de conversation, juste l'affichage visuel.

---

### `/plugin`

Gère les plugins installés dans le dossier `plugins/`.

```
/plugin                       → identique à /plugin list
/plugin list                  → liste tous les plugins trouvés, actifs ou non
/plugin on <nom>               → active un plugin
/plugin off <nom>              → désactive un plugin
/plugin download <lien>        → télécharge et installe un plugin depuis une archive ZIP
/plugin uninstall <nom>        → désactive puis supprime définitivement un plugin du disque
```

Le `<nom>` est le nom du **dossier** du plugin (pas forcément le `display_name`).

Exemple d'affichage de `/plugin list` :
```
┌─ plugins disponibles ────────────────────────
│ ● couleur — Ajoute /color...
│ ○ personnalisation — Ajoute /enchai...
│ ○ meteo — Prévisions météo  [1 dépendance(s)]
│
│ /plugin on <nom>  /plugin off <nom>  /plugin download <lien>  /plugin uninstall <nom>
└───────────────────────────────────────────────
```

`●` = actif, `○` = inactif. Le tag `[N dépendance(s)]` apparaît si le plugin déclare des paquets pip requis (voir [section 9.5](#95--déclarer-des-dépendances-pip)).

L'activation est **persistante** : un plugin activé reste activé aux prochains lancements de Stelar, jusqu'à ce que tu fasses `/plugin off`.

#### `/plugin download <lien>`

Télécharge un plugin depuis un **lien de téléchargement direct d'une archive ZIP** (typiquement un export GitHub type `https://github.com/user/repo/archive/refs/heads/main.zip`, ou tout lien HTTP renvoyant directement un fichier `.zip`).

Ce que fait la commande, dans l'ordre :

1. Télécharge l'archive (limite de sécurité : 50 Mo max)
2. Vérifie que c'est un ZIP valide
3. Vérifie qu'aucun fichier de l'archive ne tente de s'extraire en dehors du dossier prévu (protection anti "zip-slip")
4. Cherche `plugin.json` + `plugin.py`, soit directement à la racine de l'archive, soit dans son unique sous-dossier (le cas classique d'un ZIP GitHub, qui enveloppe tout dans `repo-branche/`)
5. Détermine le nom du dossier final à partir du champ `name` de `plugin.json`
6. Si un plugin du même nom existe déjà, demande confirmation avant de l'écraser (et le désactive proprement s'il était actif)
7. Copie le plugin dans `plugins/<nom>/`

```
/plugin download https://github.com/quelquun/stelar-meteo/archive/refs/heads/main.zip
```

Une fois téléchargé, le plugin **n'est pas activé automatiquement** — utilise `/plugin on <nom>` pour l'activer (et installer ses dépendances si besoin).

#### `/plugin uninstall <nom>`

Demande confirmation, désactive le plugin s'il est actif, puis **supprime définitivement son dossier** sous `plugins/`. Action irréversible — le code du plugin est perdu, pas juste désactivé.

---

### `/exit`, `/quit`

Quitte proprement Stelar. Sauvegarde l'historique de saisie (flèches haut/bas) avant de fermer.

---

## 6. Backends IA : Cloud vs Local

Stelar peut parler à deux types de serveurs Ollama, configurables via `/settings` :

### Ollama Cloud

- URL fixe : `https://ollama.com/v1/chat/completions`
- Nécessite une **clé API** (`/config <clé>` ou via `/settings`)
- Modèle par défaut : `gemma4:31b-cloud`
- Format de réponse : compatible OpenAI (flux `data: {...}` en JSON)

### Ollama Local

- URL construite : `http://<host>:<port>/api/chat`
- **Aucune clé API requise**
- Host par défaut : `localhost`
- **Port modifiable manuellement** — par défaut `11434` (le port standard d'Ollama), mais tu peux mettre n'importe quel port si ton serveur local écoute ailleurs
- Modèle par défaut : `llama3.1` (à adapter selon ce que tu as téléchargé avec `ollama pull`)
- Format de réponse : natif Ollama (un objet JSON complet par ligne, se terminant par `{"done": true}`)

**Basculer entre les deux** se fait uniquement via `/settings` → étape 2. Stelar gère en interne les deux formats de flux différents, tu n'as rien à adapter côté usage — le chat fonctionne pareil quel que soit le backend actif.

Si le backend local est sélectionné mais que le serveur ne répond pas (pas lancé, mauvais port...), Stelar affiche une erreur claire :
```
✖ Impossible de contacter Ollama local sur http://localhost:11434.
  Vérifiez qu'il tourne (`ollama serve`) et que le port est correct (/settings).
```

---

## 7. Le système d'actions (write_file, execute...)

L'IA peut demander à Stelar d'agir sur ton système de fichiers ou d'exécuter des commandes shell. Elle le fait en incluant un bloc spécial dans sa réponse :

```
<<<ACTION>>>
{"action": "write_file", "path": "notes.txt", "content": "Mes notes"}
<<</ACTION>>>
```

**Actions disponibles :**

| Action | Paramètres | Effet |
|---|---|---|
| `write_file` | `path`, `content` | Écrit (ou écrase) un fichier |
| `read_file` | `path` | Lit un fichier et affiche son contenu (tronqué à 2000 caractères) |
| `list_dir` | `path` | Liste le contenu d'un dossier |
| `execute` | `command` | Exécute une commande shell (délai max 30s) |

**Sécurité :** dès que l'IA propose une ou plusieurs actions, Stelar les affiche toutes dans une boîte encadrée avant de rien exécuter :

```
┌─ 2 action(s) demandée(s) ─────────────────────
│ write_file  →  notes.txt
│ execute  →  ls -la
└─────────────────────────────────────────────
  ? Exécuter ces actions ? (o/N) :
```

Rien ne s'exécute sans ta confirmation explicite (`o`), sauf si `auto_approve_actions` est activé à `true` dans `config.json` (non exposé par une commande — à modifier manuellement dans le fichier si tu es sûr de ce que tu fais).

---

## 8. Le fichier config.json en détail

Voici toutes les clés que contient `storage/config.json`, généré et maintenu automatiquement :

| Clé | Type | Rôle |
|---|---|---|
| `api_key` | texte | Clé API du backend cloud |
| `model` | texte | Modèle utilisé en backend cloud |
| `user_name` | texte | Ton nom affiché dans le prompt |
| `ai_name` | texte | Nom affiché devant les réponses de l'IA |
| `system_prompt` | texte | Instructions système envoyées à chaque requête |
| `personalization` | texte | Texte additionnel ajouté au prompt système (généré par le plugin `personnalisation` via `/enchai`, ou modifiable à la main) |
| `auto_approve_actions` | booléen | Si `true`, les actions système s'exécutent sans confirmation |
| `backend` | `"cloud"` ou `"local"` | Backend IA actif |
| `local_host` | texte | Adresse du serveur Ollama local |
| `local_port` | nombre | Port du serveur Ollama local |
| `local_model` | texte | Modèle utilisé en backend local |
| `theme` | objet | Couleurs actives, une entrée `[r, g, b]` par cible (voir `/theme`) |
| `input_mode` | `"single"` ou `"multi"` | Mode de saisie actif |

Tu peux éditer ce fichier à la main quand Stelar est **fermé** — au prochain lancement, les clés manquantes sont recomplétées avec leurs valeurs par défaut, et un JSON corrompu est automatiquement réinitialisé plutôt que de faire planter le programme.

---

## 9. Construire un plugin

Un plugin ajoute une ou plusieurs commandes `/xxx` à Stelar. Deux fichiers minimum, dans un dossier sous `plugins/` :

```
plugins/
└── mon_plugin/
    ├── plugin.json
    └── plugin.py
```

Le **nom du dossier** est l'identifiant utilisé par `/plugin on mon_plugin`.

### 9.1 — `plugin.json`

```json
{
  "name": "mon_plugin",
  "display_name": "Mon Super Plugin",
  "description": "Ce que fait le plugin, affiché dans /plugin list",
  "version": "1.0",
  "commands": ["/macommande"],
  "dependencies": ["requests", "beautifulsoup4==4.12.0"]
}
```

Champs utilisés par le core :
- `name` — utilisé comme nom de dossier de destination lors d'un `/plugin download`
- `description` — affiché dans `/plugin list`
- `dependencies` — liste de paquets pip requis, installés automatiquement à l'activation (voir [9.5](#95--déclarer-des-dépendances-pip))

`display_name`, `version` et `commands` sont là pour la documentation humaine du plugin — le core ne les lit pas directement.

### 9.2 — `plugin.py`

Le fichier **doit** définir une fonction `register(api)` qui retourne un dictionnaire `{"/commande": fonction}`.

```python
def register(api):
    def cmd_hello(args):
        nom = api.ask("Quel est ton nom ?")
        api.print_success(f"Salut {nom} !")

    return {
        "/hello": cmd_hello,
    }
```

Chaque fonction de commande reçoit `args`, la liste des mots tapés après la commande. Exemple : `/hello foo bar` → `args = ["foo", "bar"]`.

### 9.3 — Activation

```
/plugin list           → vérifie que ton plugin apparaît (scan automatique du dossier plugins/)
/plugin on mon_plugin   → l'active, persiste entre les sessions
/plugin off mon_plugin  → le désactive
```

Aucun redémarrage nécessaire : dès que le dossier existe avec les deux fichiers, `/plugin list` le détecte.

### 9.4 — Règles importantes

- Ne jamais laisser une exception non gérée s'échapper de `register()` — utilise `try/except` à l'intérieur de tes commandes. Si `register()` lève une exception, le plugin refuse de charger et Stelar affiche l'erreur, mais continue de fonctionner normalement.
- Les noms de commandes doivent commencer par `/`.
- Si deux plugins définissent la même commande, le dernier chargé (ordre alphabétique des dossiers, ou ordre d'activation) écrase le premier dans la table de commandes.
- Ne jamais faire `import stelar` directement dans un plugin — ça échoue selon la façon dont `stelar.py` a été lancé (nom de module `__main__` vs `stelar`). Utilise toujours `api.core`, qui te donne la même référence de façon fiable (voir section suivante).

### 9.5 — Déclarer des dépendances pip

Si ton plugin a besoin de paquets Python qui ne font pas partie de la bibliothèque standard, déclare-les dans `plugin.json` :

```json
{
  "name": "meteo",
  "description": "Prévisions météo en direct",
  "dependencies": ["requests", "python-dateutil>=2.8"]
}
```

Au moment où l'utilisateur fait `/plugin on meteo` :

1. Stelar vérifie quels paquets de la liste sont déjà importables
2. S'il en manque, il affiche la liste et **demande confirmation** avant d'installer quoi que ce soit
3. Chaque paquet manquant est installé via `pip install` (avec repli automatique si `--break-system-packages` n'est pas supporté par l'environnement, ce qui couvre Termux comme les venv classiques)
4. Si une installation échoue, l'activation du plugin est annulée et l'erreur pip est affichée
5. Une fois toutes les dépendances confirmées présentes, le plugin est chargé normalement

Aucune action n'est nécessaire dans `plugin.py` lui-même — importe simplement le paquet normalement dans tes fonctions de commande, il sera disponible.

**Format accepté** : n'importe quel spécificateur pip valide (`"requests"`, `"requests==2.31.0"`, `"requests>=2.0,<3.0"`...).

### 9.6 — Distribuer ton plugin via `/plugin download`

Pour que d'autres utilisateurs installent ton plugin avec `/plugin download <lien>`, il suffit que le lien pointe vers une **archive ZIP téléchargeable directement** contenant `plugin.json` et `plugin.py` — à la racine de l'archive, ou dans un unique sous-dossier (c'est automatiquement le cas si tu utilises le lien "Download ZIP" d'un dépôt GitHub, par exemple `https://github.com/toi/ton-plugin/archive/refs/heads/main.zip`).

Aucune structure ou métadonnée supplémentaire n'est nécessaire — Stelar détecte la racine du plugin, lit `name` dans `plugin.json` pour nommer le dossier d'installation, et gère lui-même les cas de conflit avec un plugin déjà installé du même nom.

---

## 10. Référence complète de l'API plugin

L'objet `api` passé à `register(api)` expose les méthodes suivantes.

### Affichage

```python
api.print_system("texte gris discret, précédé d'un ·")
api.print_success("texte vert, précédé d'un ✔")
api.print_error("texte rouge, précédé d'un ✖")
api.print_warn("texte jaune, précédé d'un !")
api.print_box("titre", ["ligne 1", "ligne 2"], color=None)
```

`print_box` encadre les lignes dans une boîte façon `┌─ titre ─...─┐`. Le paramètre `color` est optionnel — sans lui, la couleur `box` du thème actif est utilisée.

### Interaction utilisateur

```python
val = api.ask("Ta question ?", default="valeur par défaut")
oui = api.ask_yes_no("Confirmer ?", default_no=True)
```

`ask` retourne la saisie de l'utilisateur, ou `default` si l'entrée est vide (ou si `Ctrl+C`/`Ctrl+D` est pressé). `ask_yes_no` retourne un booléen ; seul `o` compte comme "oui".

### Configuration persistante

```python
config = api.get_config()          # dict (référence directe à la config en mémoire)
config["ma_cle"] = "valeur"
api.save_config()                  # écrit sur disque (storage/config.json)
```

`get_config()` retourne le dictionnaire de config **en mémoire** — le modifier modifie directement l'objet utilisé par tout Stelar. `save_config()` doit être appelé explicitement pour persister sur disque.

### Appeler l'IA sans polluer le chat visible

```python
texte = api.ask_ai("Résume ceci : ...", system="Tu es concis.")
```

Envoie une requête **non-streamée** au backend actuellement actif (cloud ou local, selon `config["backend"]`), affiche un spinner "Génération" pendant l'attente, et retourne le texte complet de la réponse — sans jamais l'afficher automatiquement dans le chat. À toi de décider quoi en faire (l'afficher via `print_box`, l'enregistrer dans la config...).

### Accès au module core

```python
stelar = api.core
```

Te donne une référence directe au module `stelar.py` en cours d'exécution, quelle que soit la façon dont il a été lancé. Utile si tu as besoin d'accéder à des éléments non exposés par l'API haut-niveau, par exemple :

```python
def register(api):
    stelar = api.core

    def cmd_exemple(args):
        # Accès direct aux couleurs actives
        print(stelar.C.FG_ACCENT)
        # Accès aux fonctions utilitaires internes
        rgb = stelar.parse_css_color("#ff0000")

    return {"/exemple": cmd_exemple}
```

**Ne fais jamais `import stelar`** directement — utilise systématiquement `api.core`.

---

## 11. Dépannage

**"Le module 'requests' est requis"**
→ `pip install requests` (ou avec `--break-system-packages` sur certains systèmes Linux récents).

**"Clé API manquante"**
→ Backend cloud actif sans clé configurée. Utilise `/config <clé>` ou passe en local via `/settings`.

**"Impossible de contacter Ollama local sur http://..."**
→ Le serveur Ollama local n'est pas lancé, ou le port configuré ne correspond pas. Lance `ollama serve` et vérifie le port avec `/settings`.

**Erreur au chargement d'un plugin**
→ Le message affiché contient l'exception Python levée. Vérifie que `plugin.py` définit bien une fonction `register(api)` qui ne lève pas d'exception, et qu'il n'y a pas d'erreur de syntaxe (`python3 -m py_compile plugin.py` pour vérifier).

**Config ou mémoire semble "réinitialisée" après une erreur**
→ Comportement volontaire : si `config.json` ou `memory.json` est corrompu (JSON invalide), Stelar le remplace silencieusement par des valeurs par défaut plutôt que de planter au démarrage. Une sauvegarde régulière du dossier `storage/` est recommandée si l'historique t'importe.

**Les couleurs ne s'affichent pas / restent grises**
→ Ton terminal ne supporte peut-être pas les couleurs, ou la variable d'environnement `NO_COLOR` est définie. Stelar désactive automatiquement la couleur dans ce cas pour rester lisible.

**"Archive rejetée : chemin dangereux détecté"**
→ `/plugin download` a détecté un fichier dans le ZIP qui tenterait de s'extraire en dehors du dossier plugin (protection anti "zip-slip"). Le téléchargement est bloqué par sécurité — ne provient probablement pas d'une source fiable.

**"Aucun plugin.json + plugin.py trouvé dans l'archive"**
→ Le ZIP téléchargé ne contient pas les deux fichiers requis, ni à sa racine ni dans un unique sous-dossier de premier niveau. Vérifie que le lien pointe bien vers l'archive complète du plugin (voir [9.6](#96--distribuer-ton-plugin-via-plugin-download)).

**Installation d'une dépendance pip qui échoue**
→ Le message affiche les dernières lignes de la sortie pip. Cause fréquente : nom de paquet incorrect dans `dependencies`, absence de connexion réseau, ou environnement Python en lecture seule (essaie `pip install <paquet>` manuellement pour voir l'erreur complète).
