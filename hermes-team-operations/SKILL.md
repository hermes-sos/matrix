---
name: hermes-team-operations
description: Règles de fonctionnement de l'équipe Hermes — tours de parole, anti-boucle, partage de skills, décisions.
version: 1.0.0
metadata:
  hermes:
    tags: [team, orchestration, rules]
---

# Hermes Team Operations

## Règle de parole
Ordre nominal : SOS → Three → Five → Chris.
Un agent ne répond que si :
- Christophe le sollicite directement (@agent)
- L'orchestrateur lui donne explicitement la parole
- Un autre agent lui passe la main nommément

**Jamais** répondre à un message qui ne t'est pas adressé.

## Anti-boucle
- Ne pas envoyer d'accusés de réception (« reçu », « j'attends », « ok »)
- Ne pas commenter les messages des autres agents sauf si sollicité
- Une réponse = un seul message. Pas de suivi automatique.
- En cas de doute, attendre une consigne explicite.

## Tâches longues
Le tour de parole ne bloque que les messages dans la room.
Un agent peut travailler en arrière-plan si Christophe lui a confié une mission.

## Skills partagés
Les skills communes sont versionnées dans un dépôt partagé.
Chaque agent les installe depuis cette source unique.

## Décision humaine
Christophe est le seul décideur. Les agents proposent, Christophe tranche.
Ne jamais déployer, modifier une config, ou lancer une action sans GO explicite.
