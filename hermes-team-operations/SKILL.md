---
name: hermes-team-operations
description: Règles de fonctionnement de l'équipe Hermes — tours de parole, anti-boucle, partage de skills, décisions.
version: 1.3.0
metadata:
  hermes:
    tags: [team, orchestration, rules, orchestrator-v3, commands]
---

# Hermes Team Operations

## Règle de parole
Ordre nominal : SOS → Three → Five → Chris.
Un agent ne répond que si :
- Christophe le sollicite directement (@agent)
- L'orchestrateur lui donne explicitement la parole
- Un autre agent lui passe la main nommément

**Jamais** répondre à un message qui ne t'est pas adressé.

**Power levels** : Dendrite enforce bien les power levels Matrix (testé et vérifié le 29/06/2026 — un utilisateur avec power level 0 est bien bloqué par `M_FORBIDDEN`).

⚠️ **Règle Matrix critique** : un utilisateur de niveau N ne peut **pas** modifier un autre utilisateur de niveau ≥ N. Donc un orchestrateur à 50 **ne peut pas muter** un agent à 50 (50 ≥ 50 → `M_FORBIDDEN`). D'où le design corrigé :

| Niveau | Qui | Effet |
|---|---|---|
| 100 | Christophe | Admin, contrôle total |
| **50** | Orchestrateur | Modérateur (peut mute 49 car 49 < 50) |
| **49** | Agents (démutés) | Peut parler (`m.room.message` exige 49) |
| 0 | Agents (mutés) | Bloqué |

Le mute/démute de l'orchestrateur a un **effet réel côté serveur**. L'anti-boucle v3 (ignorer les messages hors tour) reste une sécurité supplémentaire. Voir `references/orchestrator-technical.md`.

## Commandes orchestrateur (`/orch`)

Christophe contrôle l'orchestrateur via 15 commandes dans la room Matrix.

| Commande | Effet |
|---|---|
| `/orch status` | Cycle, speaker, ordre, file d'attente |
| `/orch stop` | Arrête le cycle en cours |
| `/orch pause` | Suspend les nouveaux cycles |
| `/orch resume` | Réactive l'orchestrateur |
| `/orch skip` | Passe au speaker suivant |
| `/orch ordre A,B,C,D` | Change l'ordre de parole |
| `/orch default-order` | Restaure l'ordre par défaut |
| `/orch agents` | Liste les agents et leur état |
| `/orch disable @agent` | Retire un agent des cycles |
| `/orch enable @agent` | Réintègre un agent |
| `/orch queue` | Affiche les questions en attente |
| `/orch clearqueue` | Vide la file d'attente |
| `/orch reset` | Reset complet du state |
| `/orch log [N]` | N dernières lignes de log |
| `/orch help` | Liste toutes les commandes |

**Important** : `disable`/`enable` retire/réintègre l'agent du cycle côté orchestrateur, indépendamment des power levels. Le mute Matrix (power level 0 → bloqué par Dendrite) reste le mécanisme de base pour empêcher les messages non sollicités ; `disable` est un cran supplémentaire qui exclut totalement l'agent des tours.

## Anti-boucle
- Ne pas envoyer d'accusés de réception (« reçu », « j'attends », « ok »)
- Ne pas commenter les messages des autres agents sauf si sollicité
- Une réponse = un seul message. Pas de suivi automatique.
- En cas de doute, attendre une consigne explicite.
- **Technique (v3)** : l'orchestrateur ignore tout message d'un agent qui n'est pas `current_speaker`. Log interne uniquement, pas de réponse publique.
- **Filtre `is_system_msg()`** : en plus de l'anti-boucle, l'orchestrateur filtre les messages contenant des patterns système (ex: `Interrupting current task`, `📬 No home channel`). Voir `references/orchestrator-technical.md` pour la liste complète.

## Déboguer sans SSH

Quand le VPS est inaccessible (clé SSH expirée, timeout) :
- Le code source de l'orchestrateur v3 est sauvegardé localement : `C:\Users\strap\hermes-sos-workspace\orchestrator_v3.py`
- Accessible en Python via `/mnt/c/Users/strap/hermes-sos-workspace/orchestrator_v3.py` (MSYS)
- Lire ce fichier permet d'analyser le cycle, les ordres, les filtres, sans connexion au VPS
- ⚠️ Cette copie peut être déphasée par rapport à la version en production — vérifier la date de dernière synchro

## Tâches longues
Le tour de parole ne bloque que les messages dans la room.
Un agent peut travailler en arrière-plan si Christophe lui a confié une mission.

## Skills partagés
Les skills communes sont versionnées dans le dépôt Git `nauticode-skills` (GitHub NautiCode).
Chaque agent les installe depuis cette source unique.

## Décision humaine
Christophe est le seul décideur. Les agents proposent, Christophe tranche.
Ne jamais déployer, modifier une config, ou lancer une action sans GO explicite.



## Pièges Matrix

### Duplication Telegram → Matrix

Quand on travaille sur l'orchestrateur ou qu'on échange en DM Telegram pendant qu'un cycle Matrix est actif :
- Le plugin Matrix d'Hermes relaye les messages Telegram vers la room `hermes-team`
- L'orchestrateur peut les détecter et créer des boucles de duplication (messages reçus 13-18 fois)
- **Solution** : désactiver temporairement Matrix dans `config.yaml` (`matrix: enabled: false`) le temps des travaux, puis `/restart` dans Telegram
- Une fois le travail terminé, réactiver (`enabled: true`) et `/restart` à nouveau

### Streaming Telegram → doublons

Si les messages Telegram apparaissent en double (liste répétitive de tool outputs) :
- **Cause** : `platforms.telegram.streaming: true` dans `config.yaml`
- **Fix** : passer à `false` → `sed -i 's/      streaming: true/      streaming: false/' config.yaml`
- **Redémarrage obligatoire** : `/restart` depuis Telegram (⚠️ tue la gateway, ne la relance pas — relancer manuellement depuis l'interface Windows ou la tâche planifiée)

### Orchestrateur : plantage `disabled_agents`

Si l'orchestrateur crash avec `NameError: name 'disabled_agents' is not defined` :
- **Cause** : `start_cycle()` référence `disabled_agents` (variable locale de `main()`) sans le recevoir en paramètre
- **Fix** : `start_cycle(question, order, admin_event_id, disabled_agents=None)` — voir `references/orchestrator.md` pour le code complet

### Orchestrateur : MUTE échoue (HTTP 403)

Si les logs montrent `MUTE @agent: 50 → 0 [FAIL]` avec erreur `old level is equal to or above the level of the sender` :
- **Cause** : l'orchestrateur est niveau 50, l'agent est niveau 50 → Matrix interdit (50 ≥ 50)
- **Fix** : baisser `m.room.message` à 49 et `unmute()` à 49 au lieu de 50. L'orchestrateur à 50 peut alors muter (49 < 50)
- **Vérification** : `grep "MUTE\|DEMUTE" /opt/dendrite/data/orch_v2.log | grep FAIL`

### Orchestrateur : power levels supprimés (M_NOT_FOUND)

Si `GET /state/m.room.power_levels` retourne 404 :
- **Cause** : un PUT invalide a corrompu l'état Dendrite (bug MUTE 50→0)
- **Fix immédiat** : recréer les power levels avec le token Christophe (admin 100) :
  ```python
  pl = {"users_default": 0, "events": {"m.room.message": 49, ...},
        "users": {"@christophe:videocours.fr": 100, "@orchestrateur:videocours.fr": 50, ...}}
  req("PUT", f"rooms/{ROOM}/state/m.room.power_levels", pl)  # avec token Christophe
  ```

L'orchestrateur lit son token depuis `os.environ["ORCH_TK"]`, pas depuis le fichier.
- **Lancement correct** : `ORCH_TK=$(cat /opt/dendrite/data/orch_token.txt) nohup python3 -u orchestrator.py >> log 2>&1 & disown`
- Ou via wrapper Python qui lit le fichier et passe l'env au subprocess
