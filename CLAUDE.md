# CLAUDE.md — Projet Hardcode

## Purpose

Framework de gestion de valeurs de paramétrage dynamique (« hardcodes ») stockées en base de données, avec cache Shared Memory (SHM), gestion de versions, transport inter-systèmes via RFC, et exécution dynamique de code source ABAP.

---

## Repository Format

ZIP abapgit (`ZHARDCODE_7.0.zip`), extrait dans `C:\Temp\hardcode_audit\src\`

---

## Object Inventory

| Type       | Nom                          | Description |
|------------|------------------------------|-------------|
| Class      | `ZCL_HARDCODE`               | Classe principale — accès aux valeurs hardcodées (Singleton par REPID) |
| Class      | `ZCL_HARDCODE_MAINTAIN`      | Maintenance (ajout/suppression versions, paramètres, attributs) |
| Class      | `ZCL_HARDCODE_CLUSTER`       | Stockage du code source en cluster DB (EXPORT/IMPORT avec compression) |
| Class      | `ZCL_HARDCODE_FILE`          | Import/Export hardcodes via fichier ZIP |
| Class      | `ZCL_HARDCODE_SHM`           | Gestion du cache Shared Memory |
| Class      | `ZCL_HARDCODE_SHM_AREA`      | Zone SHM (générée, hérite de `CL_SHM_AREA`) |
| Class      | `ZCL_HARDCODE_SHM_ROOT`      | Racine SHM (données hardcodes en mémoire partagée) |
| Class      | `ZCL_DYNAMIC_SOURCE_CODE`    | Exécution dynamique de code ABAP (GENERATE SUBROUTINE POOL) |
| Class      | `ZCL_GUI_ALV_TREE_UTIL`      | Utilitaire ALV Tree |
| Class      | `ZCL_ITAB_HISTORY`           | Gestion historique d'internal tables |
| Class      | `ZCL_TEXT_UTILITIES`         | Utilitaires texte |
| Exception  | `ZCX_HARDCODE`               | Exception principale |
| Exception  | `ZCX_HARDCODE_MAINTAIN`      | Exception maintenance |
| Exception  | `ZCX_HARDCODE_SHM`           | Exception SHM |
| FG         | `ZTEC_HARDCODE`              | Function Group |
| FM         | `Z_HC_READ`                  | Lecture hardcodes (RFC-enabled) |
| FM         | `Z_HC_TRANSPORT`             | Transport hardcodes (RFC-enabled) |
| FM         | `Z_HC_SHM_RELOAD`            | Rechargement SHM (RFC-enabled) |
| FM         | `ZH_VALUE_HELP_HARDCODE`     | Aide à la saisie |
| Program    | `ZTEC_HARDCODE`              | Programme de maintenance IHM (ALV Tree) |
| Program    | `ZTEC_HARDCODE_XPRA_TRANSPORT` | Programme XPRA transport |
| Program    | `Z_BC_HC_SOURCE_EXEC`        | Exécution de code source dynamique via Hardcode |

---

## Audit Scope

Audit complet des 8 catégories selon la méthodologie root (naming, error handling, performance, security, dead code, hardcode, modernization, inconsistency).

---

## Audit Status

Voir `MEMORY.md` et `TODO.md` dans ce dossier.
