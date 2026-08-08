# Architecture — contrats et modèle de sécurité

Ce document détaille le fonctionnement interne du pipeline. Pour l'installation
et l'usage courant, voir le [README](../README.md).

## Pourquoi des contrats de fichiers

Les quatre agents ne se transmettent pas d'information en conversation. Chacun
écrit un artefact JSON sur disque, que le suivant relit. Ce choix a trois
conséquences directes :

- **Une étape qui échoue est visible.** L'orchestrateur vérifie l'existence du
  fichier de sortie après chaque phase et s'arrête avec un message explicite s'il
  est absent, au lieu de laisser l'agent suivant travailler sur du vide.
- **Chaque exécution est inspectable.** Les artefacts restent dans
  `.claude/quality-team/` après le run. Un désaccord sur une violation se
  tranche en ouvrant `violations.json`, pas en relançant l'analyse.
- **Les agents sont remplaçables.** Tout agent respectant le schéma d'entrée et
  de sortie peut se substituer à celui fourni.

Les schémas se trouvent dans [`quality-team/schemas/`](../quality-team/schemas/).

## Les trois contrats

### `findings.json` — scout → principles-auditor

Des faits mesurés, sans jugement qualitatif.

| Champ | Type | Contenu |
|---|---|---|
| `scope` | string | Chemin analysé |
| `generated_at` | string | Timestamp ISO |
| `project` | object | `manifests`, `languages`, `frameworks`, `playbooks_applicable`, `validation_commands` |
| `tools_used` | string[] | Outils réellement exécutés |
| `validation_candidates` | validationCommand[] | Commandes détectées |
| `hotspots` | object[] | `file`, `score`, `reason`, `blast_radius` — triés par score décroissant, 20 maximum |
| `dead_code` | object[] | `file`, `symbol`, `type`, `tool` — uniquement si confirmé par un outil cross-fichiers |
| `complexity` | object[] | `file`, `fn`, `ccn`, `nloc`, `params`, `threshold_breached` |
| `clones` | object[] | `files`, `lines`, `tool` |
| `lint` | object[] | `file`, `line`, `rule`, `severity` (`error` \| `warn` \| `info`), `message`, `tool` |
| `unused_deps` | object[] | `package`, `manager`, `type`, `tool` |
| `errors` | object[] | `tool`, `error` — outils absents, timeouts, échecs de parsing |

Le champ `errors` est structurant : un outil indisponible n'interrompt pas le
scout, il produit une entrée et l'analyse continue. C'est ce qui permet au
pipeline de fonctionner sans aucun outillage externe.

`$defs.validationCommand` : `name`, `command`, `reason`, `applies_to`
(les trois premiers requis).

### `violations.json` — principles-auditor → refactor-executor

Des jugements classés, chacun rattaché à un principe et à une preuve.

| Champ | Type | Contenu |
|---|---|---|
| `generated_at` | string | Timestamp ISO |
| `files_analyzed` | integer | Nombre de fichiers réellement lus |
| `blocking` | violation[] | Bug probable, perte de données, erreur silencieuse, sécurité, contrat externe cassé |
| `important` | violation[] | Dette significative, complexité forte, duplication structurante |
| `nit` | violation[] | Style, nommage mineur, documentation simple |
| `suggestion` | violation[] | Amélioration optionnelle ou incertaine |
| `ai_smells` | object[] | `file`, `rule`, `description`, `severity` |
| `manual_verify` | object[] | `file`, `reason`, `recommendation` |

`$defs.violation` : `file`, `line`, `principle`, `description`, `evidence`,
`fix_hint` — `file`, `principle`, `description` et `fix_hint` sont requis.
Une violation sans piste de correction n'est donc pas représentable.

Deux règles gouvernent le classement :

- **L'incertitude descend.** En cas de doute, l'auditeur classe en `suggestion`,
  jamais en `blocking`.
- **`manual_verify` est exclusif.** Un fichier placé en `manual_verify` ne peut
  pas figurer simultanément comme candidat automatique en `blocking` ou
  `important`. C'est ce qui empêche l'exécuteur de toucher un fichier que
  l'auditeur a jugé trop risqué.

### `changes.json` — refactor-executor → doc-updater

Ce qui a été fait, ce qui ne l'a pas été, et pourquoi.

| Champ | Type | Contenu |
|---|---|---|
| `generated_at` | string | Timestamp ISO |
| `applied` | object[] | `file`, `type`, `description`, `blast_radius_checked`, `validated`, `tools_passed` — tous requis |
| `skipped` | object[] | `file`, `reason` requis ; `validation_output`, `recommendation`, `blast_radius` optionnels |
| `validation_results` | validationResult[] | `name`, `command`, `status` (`pass` \| `fail` \| `skipped`), `output_path`, `summary` |

Le fait que `validated` et `tools_passed` soient requis sur chaque entrée
`applied` signifie qu'un changement appliqué sans validation possible est
enregistré comme tel (`validated: false`, `tools_passed: []`) plutôt que présenté
comme vérifié.

## Déroulé

| Phase | Acteur | Entrée | Sortie |
|---|---|---|---|
| **0** — Détection | orchestrateur | manifests, extensions du scope | `project_profile.json`, `validation_commands.json`, `baseline_validation.json` |
| **0b** — Références | orchestrateur | `references/`, `playbooks/` | prompts enrichis |
| **1** — Cartographie | `scout` | scope, profil projet | `findings.json` |
| **2** — Audit | `principles-auditor` | `findings.json`, références | `violations.json` |
| **2b** — Plan | orchestrateur | `findings`, `violations`, `validation_commands` | `refactor_plan.md` + **approbation utilisateur** |
| **3** — Exécution | `refactor-executor` | plan approuvé | `changes.json` |
| **4** — Rapport | `doc-updater` | tous les artefacts | `REFACTOR_REPORT.md` |
| **5** — Synthèse | orchestrateur | `baseline_validation.json` vs post-refactor | comparaison affichée |

La phase 0 enregistre une **baseline de validation** avant toute modification.
La phase 5 rejoue les mêmes commandes et affiche la comparaison avant → après.
Sans cette baseline, un test déjà rouge avant le run serait imputé au
refactoring.

La phase 3 est sautée si `mode = audit-only`, ou si le plan n'a pas reçu
d'approbation explicite. En mode `audit-only`, le plan est présenté à titre de
recommandation.

## Modèle de sécurité

### Permissions par agent

Les restrictions sont déclarées dans le frontmatter de chaque agent, donc
appliquées par Claude Code et non par la consigne textuelle.

| Agent | `tools` | Écriture possible |
|---|---|---|
| `scout` | `Read`, `Glob`, `Grep`, `Bash` | aucune |
| `principles-auditor` | `Read`, `Grep` | aucune — pas même `Bash` |
| `refactor-executor` | `Read`, `Write`, `Edit`, `MultiEdit`, `Bash` | code source, dans le périmètre du plan |
| `doc-updater` | `Read`, `Write`, `Edit` | documentation uniquement, pas de `Bash` |

Le skill orchestrateur lui-même est limité à `Read` et `Bash` : il ne peut pas
modifier de fichier directement.

### Échelle de risque

[`references/safe-refactor.md`](../quality-team/references/safe-refactor.md)
répartit les opérations en quatre niveaux :

1. **Toujours sûr après confirmation outillage** — suppression de code mort
   confirmé, de logs de debug non fonctionnels, de blocs commentés de plus de
   3 lignes ; extraction d'une constante nommée dans le même fichier ;
   correction de documentation publique sans toucher signature ni logique.
2. **Sûr avec validation post-modification** — renommage avec mise à jour de
   tous les usages confirmés, extraction d'une petite fonction locale,
   déplacement d'une fonction interne, simplification d'une condition. Si aucune
   validation n'est disponible, ces opérations sont proposées dans le plan mais
   traitées comme un risque supérieur.
3. **Jamais sans validation humaine séparée** — signature ou type de retour
   public, suppression de fichier entier, authentification / autorisation /
   chiffrement / secrets / sessions, migrations et schémas de base de données,
   tests au-delà d'un import nécessaire à un rename, configuration de build ou
   de CI, nouvelle dépendance, API externe.
4. **Jamais touché** — fichiers générés ou vendorés, répertoires de build
   (`dist/`, `build/`, `target/`, `.next/`, `out/`, `.cache/`, `vendor/`),
   lockfiles, fichiers marqués `DO NOT EDIT` ou `GENERATED`, et tout fichier
   listé dans `violations.manual_verify`.

### Protocole par fichier

L'exécuteur applique la même séquence à chaque fichier, et s'arrête à la
première condition bloquante :

1. Vérifier la blacklist et `manual_verify`.
2. Vérifier le blast radius via Qartez si le fichier est un hotspot.
3. Lire le fichier, identifier le plus petit changement suffisant.
4. N'appliquer que ce qui figure dans `refactor_plan.md`.
5. Lancer les validations applicables.
6. **Si une validation échoue, revert du seul fichier touché** et journalisation
   du skip.
7. Passer au fichier suivant.

Le revert est délibérément à granularité fichier : un échec n'annule pas les
changements déjà validés sur les autres fichiers.

### Raisons de skip normalisées

`manual-verify` · `blacklisted` · `blast-radius-too-high` ·
`not-in-approved-plan` · `missing-safe-refactor-rules` · `validation-unavailable` ·
`reverted:validation-failed`

Le cas `missing-safe-refactor-rules` mérite d'être noté : si les règles de
sécurité ne sont pas parvenues à l'exécuteur, celui-ci n'applique **aucun**
changement et place tous les candidats dans `skipped`. L'absence de garde-fou
n'est pas traitée comme une absence de contrainte.

## Étendre le pipeline

### Ajouter un playbook

Un playbook est un fichier de `quality-team/playbooks/` chargé sous condition.
Trois points d'accroche :

1. Créer `quality-team/playbooks/<stack>.md`, en croisant les règles avec les
   principes universels (`P1`–`P10`) plutôt qu'en les redéfinissant.
2. Déclarer la condition de chargement dans la phase 0b de
   [`SKILL.md`](../quality-team/SKILL.md).
3. S'assurer que le scout renseigne le stack dans
   `findings.project.playbooks_applicable` — l'auditeur ne charge que les
   playbooks qui y figurent.

La contrainte à respecter : **un playbook non applicable ne doit produire
aucune violation.** C'est ce qui garantit qu'ajouter un playbook ne dégrade pas
l'analyse des projets qui ne le concernent pas.

### Ajouter une règle universelle

Les règles applicables à tout langage vont dans
[`references/principles.md`](../quality-team/references/principles.md) (principes
structurants) ou
[`references/ai-smells.md`](../quality-team/references/ai-smells.md) (patterns de
code généré). Aucune modification de `SKILL.md` n'est nécessaire : ces deux
fichiers sont chargés systématiquement.

Une règle ne devient actionnable automatiquement que si elle est aussi couverte
par `safe-refactor.md`. Sinon, elle produit des constats que l'exécuteur laissera
en `skipped` avec la raison `not-in-approved-plan`.
