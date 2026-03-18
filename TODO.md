# TODO.md — Audit Hardcode

> Audit réalisé le 2026-03-17. Aucune modification de source — analyse et recommandations uniquement.

---

## 🔴 Critical

### HC-001 — `ZCL_HARDCODE_CLUSTER` : Double assignment de `attr1` dans `SOURCE_CODE_GET`
**Fichier :** `zcl_hardcode_cluster.clas.abap` ligne 265
**Code :**
```abap
ls_textid-attr1 = sy-msgv1. ls_textid-attr2 = sy-msgv2.
ls_textid-attr1 = sy-msgv3. ls_textid-attr2 = sy-msgv3.  " BUG
```
**Problème :** `attr1` et `attr2` sont réassignés avec `sy-msgv3` (deux fois). `attr3` et `attr4` ne sont jamais renseignés. Le message d'erreur sera corrompu.
**Correction :** Remplacer la ligne 265 par `ls_textid-attr3 = sy-msgv3. ls_textid-attr4 = sy-msgv4.`

---

### HC-002 — `ZCL_HARDCODE` : Accès table sans protection dans `DATA_LOAD`
**Fichier :** `zcl_hardcode.clas.abap` ligne 573
**Code :**
```abap
me->mv_version_primary = me->mt_hardcode[ version_primary = abap_true ]-version_id.
```
**Problème :** Si aucune version avec `version_primary = abap_true` n'existe, exception `CX_SY_ITAB_LINE_NOT_FOUND` non catchée → dump runtime.
**Correction :** Encapsuler dans un `TRY/CATCH cx_sy_itab_line_not_found` et lever `ZCX_HARDCODE` avec message approprié.

---

## 🟠 Major

### HC-003 — `ZCL_HARDCODE_CLUSTER` : Exceptions silencieusement ignorées dans `SOURCE_CODE_ADD` et `SOURCE_CODE_GET`
**Fichier :** `zcl_hardcode_cluster.clas.abap` lignes 155-158, 267-270, 287-289
**Problème :** Les `RAISE EXCEPTION TYPE zcx_hardcode_cluster` sont commentés dans les trois CATCH. Les erreurs de compression/décompression/écriture cluster sont avalées sans remontée. L'appelant ne peut pas savoir si l'opération a échoué.
**Correction :** Décommenter ou remplacer par une remontée via `ZCX_HARDCODE` / `ZCX_HARDCODE_MAINTAIN`. La classe `ZCX_HARDCODE_CLUSTER` semble avoir été abandonnée — utiliser `ZCX_HARDCODE_MAINTAIN`.

---

### HC-004 — `ZCL_HARDCODE_CLUSTER` : Dépendance externe non documentée sur `ZCL_B2B_INVOICE_CLUSTER`
**Fichier :** `zcl_hardcode_cluster.clas.abap` ligne 102
**Code :**
```abap
lo_abap_zip->add(
    name    = zcl_b2b_invoice_cluster=>mc_zip_content_name
    content = lv_source_code
).
```
**Problème :** `ZCL_B2B_INVOICE_CLUSTER` appartient à un autre projet (B2B). Cette dépendance crée un couplage inattendu entre projets. Le projet Hardcode ne peut pas être déployé seul.
**Correction :** Définir une constante locale `lc_zip_content_name` dans `ZCL_HARDCODE_CLUSTER` (ex : `'source_code'`).

---

### HC-005 — `ZCL_HARDCODE_SHM` : Offset `mv_key+3` hardcodé pour skiper le mandant
**Fichier :** `zcl_hardcode_shm.clas.abap` lignes 514 et 671
**Code :**
```abap
iv_repid = CONV repid( me->mv_key+3 ).  "On se positionne après le mandant
```
**Problème :** Le mandant SAP est toujours sur 3 caractères (T000-MANDT TYPE MANDT = CHAR3), mais cette logique est fragile et non documentée comme invariant. Si la clé SHM change de format, c'est silencieusement cassé.
**Correction :** Stocker `iv_repid` dans un attribut dédié (`mv_repid TYPE sy-repid`) plutôt que de recalculer par offset.

---

### HC-006 — `ZCL_DYNAMIC_SOURCE_CODE` : Whitelist d'instructions désactivée
**Fichier :** `zcl_dynamic_source_code.clas.abap` lignes 377-398
**Problème :** Le contrôle de whitelist des instructions ABAP est entièrement commenté. Seul le check SE38 + `S_DEVELOP/$TMP` subsiste. Tout utilisateur ayant accès SE38 et développement local peut exécuter n'importe quel code ABAP.
**Correction :** Implémenter ou activer le contrôle whitelist via `CL_ABAP_DYN_PRG=>CHECK_WHITELIST_TAB`.

---

### HC-007 — `ZCL_DYNAMIC_SOURCE_CODE` : `AUTHORITY-CHECK` sur `$TMP` uniquement
**Fichier :** `zcl_dynamic_source_code.clas.abap` ligne 358
**Code :**
```abap
AUTHORITY-CHECK OBJECT 'S_DEVELOP'
  ID 'DEVCLASS' FIELD '$TMP'
  ID 'OBJTYPE'  FIELD 'PROG'
```
**Problème :** Vérifier uniquement `$TMP` (package local) est insuffisant. Un utilisateur peut avoir accès à `$TMP` sans être un développeur autorisé en production.
**Correction :** Supprimer la restriction `DEVCLASS = '$TMP'` ou utiliser `DUMMY` ; ajouter un objet d'autorisation dédié `Z_HC_EXEC`.

---

### HC-008 — `Z_BC_HC_SOURCE_EXEC` : `REPID` hardcodé au lieu de `sy-repid`
**Fichier :** `z_bc_hc_source_exec.prog.abap` ligne 377
**Code :**
```abap
go_hardcode = zcl_hardcode=>get_instance( 'Z_BC_HC_SOURCE_EXEC' ).
```
**Problème :** Le nom du programme est hardcodé. Si le programme est copié/renommé, la référence est fausse. `sy-repid` est disponible dans `START-OF-SELECTION`.
**Correction :** `zcl_hardcode=>get_instance( sy-repid )`.

---

### HC-009 — `ZTEC_HARDCODE_TOP` : `REPORT` avec mauvais nom
**Fichier :** `ztec_hardcode_top.prog.abap` ligne 22
**Code :**
```abap
REPORT ztec_customizing.
```
**Problème :** Le nom déclaré dans la directive `REPORT` est `ztec_customizing`, alors que le programme cadre est `ZTEC_HARDCODE`. Incohérence de registre SAP.
**Correction :** `REPORT ztec_hardcode.`

---

### HC-010 — `ZCL_HARDCODE_SHM` : `attach_for_read( )` sans `inst_name` dans `_RELOAD`
**Fichier :** `zcl_hardcode_shm.clas.abap` lignes 729 et 753
**Code :**
```abap
me->mo_attach_read = zcl_hardcode_shm_area=>attach_for_read( ).
```
**Problème :** Sans `inst_name`, l'attachement se fait sur l'instance par défaut (`DEFAULT_INSTANCE`) et non sur `me->mv_key`. Risque de lire les hardcodes d'un autre programme.
**Correction :** `attach_for_read( CONV shm_inst_name( me->mv_key ) )`.

---

## 🟡 Minor

### HC-011 — `ZCL_DYNAMIC_SOURCE_CODE` : Faute d'orthographe dans le nom de méthode
**Fichier :** `zcl_dynamic_source_code.clas.abap`
**Code :** `_autorization_check` → devrait être `_authorization_check`
**Impact :** Nommage incohérent, difficulté de recherche.

---

### HC-012 — `ZCL_HARDCODE` : Typo dans field-symbol `<lfs_s_harcode>`
**Fichier :** `zcl_hardcode.clas.abap` ligne 173
**Code :** `<lfs_s_harcode>` → devrait être `<lfs_s_hardcode>` (double 'd' manquant)
**Impact :** Lisibilité, risque de confusion.

---

### HC-013 — `ZTEC_HARDCODE_TOP` : Variable globale `ok_code` sans préfixe `gv_`
**Fichier :** `ztec_hardcode_top.prog.abap` ligne 125
**Code :** `DATA : ok_code TYPE sy-ucomm.`
**Correction :** `DATA : gv_ok_code TYPE sy-ucomm.` (et mise à jour dans les écrans).

---

### HC-014 — `Z_BC_HC_SOURCE_EXEC` : Constantes locales avec préfixe `gc_` dans `START-OF-SELECTION`
**Fichier :** `z_bc_hc_source_exec.prog.abap` lignes 317-319
**Code :**
```abap
CONSTANTS : gc_icon_source_code TYPE icon_d VALUE '@9U@'.
CONSTANTS : gc_separation_process TYPE string VALUE '- - - ...'.
```
**Problème :** Ces constantes sont déclarées dans `START-OF-SELECTION` (contexte local), elles devraient donc utiliser le préfixe `lc_`.

---

### HC-015 — `ZCL_HARDCODE_FILE` : En-tête de méthode incorrect
**Fichier :** `zcl_hardcode_file.clas.abap` ligne 406-407
**Problème :** L'en-tête du commentaire de `_HARDCODE_GET_FROM_FILE` indique `ZCL_HARDCODE_MAINTAIN` au lieu de `ZCL_HARDCODE_FILE`.

---

### HC-016 — `ZCL_HARDCODE_SHM` : En-tête de méthode incorrect
**Fichier :** `zcl_hardcode_shm.clas.abap` ligne 387
**Problème :** Le commentaire d'en-tête de `HARDCODE_GET` indique `CONSTRUCTOR`.

---

### HC-017 — `Z_BC_HC_SOURCE_EXEC` : Faute de frappe dans la classe de message
**Fichier :** `z_bc_hc_source_exec.prog.abap` ligne 388
**Code :** `MESSAGE s001(zharcode)` → devrait être `MESSAGE s001(zhardcode)`
**Impact :** Message non trouvé à l'exécution → dump possible.

---

### HC-018 — `ZTEC_HARDCODE_XPRA_TRANSPORT` : Valeur hardcodée `'S'` pour filtre mandant
**Fichier :** `ztec_hardcode_xpra_transport.prog.abap` ligne 121
**Code :** `AND cccategory NE 'S'`
**Recommandation :** Utiliser la constante SAP `'S'` est acceptable ici (type mandant Sandbox = valeur fixe SAP standard), mais documenter l'intention.

---

### HC-019 — `ZCL_DYNAMIC_SOURCE_CODE` : Textes de message hardcodés en français
**Fichier :** `zcl_dynamic_source_code.clas.abap` lignes 102, 170
**Code :**
```abap
rs_result-execution_properties-message = 'Instruction(s) / Utilisateur non autorisé'.
|Erreur exécution : { lo_cx_root->if_message~get_longtext( ) }|
```
**Recommandation :** Utiliser la message class `ZHARDCODE`.

---

### HC-020 — `ZCL_DYNAMIC_SOURCE_CODE` : Classe locale `LCL_MAIN` hardcodée
**Fichier :** `zcl_dynamic_source_code.clas.abap` ligne 157
**Code :** `|\\PROGRAM=| && lv_name && |\\CLASS=LCL_MAIN|`
**Problème :** Si le code dynamique n'encapsule pas dans `LCL_MAIN`, l'appel échoue silencieusement.
**Recommandation :** Documenter cette contrainte d'interface, ou définir une constante `lc_main_class = 'LCL_MAIN'`.

---

### HC-021 — Dead code : méthode `CODE_INJECTION` vide mais publique
**Fichier :** `zcl_hardcode.clas.abap` lignes 77, 340-391
**Problème :** Méthode déclarée en public section mais non implémentée (entièrement commentée).
**Recommandation :** Soit implémenter, soit supprimer de la définition publique et placer en commentaire dans la section privée.

---

### HC-022 — Dead code : code commenté dans `ZCL_DYNAMIC_SOURCE_CODE._source_code_format`
**Fichier :** `zcl_dynamic_source_code.clas.abap` lignes 571-614
**Problème :** ~45 lignes d'une ancienne implémentation entièrement commentées. Nuit à la lisibilité.
**Recommandation :** Supprimer le code commenté (il est dans Git/abapgit de toute façon).

---

### HC-023 — Dead code : bloc commenté dans `ZCL_HARDCODE_SHM._reload`
**Fichier :** `zcl_hardcode_shm.clas.abap` lignes 765-772
**Problème :** `FREE` et `CREATE OBJECT` commentés, vestiges d'une ancienne implémentation.

---

### HC-024 — `ZCL_HARDCODE` : `CATCH cx_root` avec handler vide dans `_hardcode_get`
**Fichier :** `zcl_hardcode.clas.abap` ligne 890-891
**Code :** `CATCH cx_sy_itab_line_not_found cx_root.` — ENDTRY sans handler
**Impact :** Erreurs de récupération du code source silencieuses.

---

### HC-025 — `ZCL_HARDCODE_SHM` : `CATCH cx_root` handler vide dans `HARDCODE_RELOAD`
**Fichier :** `zcl_hardcode_shm.clas.abap` ligne 516
**Problème :** Échec du RFC `Z_HC_SHM_RELOAD` sur un AS ignoré sans log ni alerte.

---

### HC-026 — `ZCL_DYNAMIC_SOURCE_CODE` : `WHEN 4` et `WHEN 8` de `GENERATE SUBROUTINE POOL` non traités
**Fichier :** `zcl_dynamic_source_code.clas.abap` lignes 174-183
**Problème :** Les codes retour 4 (warning) et 8 (erreur grave) de `GENERATE SUBROUTINE POOL` ne déclenchent aucun traitement visible.

---

## 🔵 Suggestion

### HC-027 — Modernisation : `CREATE OBJECT` → `NEW #(...)`
**Fichiers :** `zcl_hardcode.clas.abap` (l.624), `zcl_hardcode_file.clas.abap` (l.104, l.131, l.497), `z_bc_hc_source_exec.prog.abap` (l.551)
**Recommandation :** Remplacer les `CREATE OBJECT ... EXPORTING` par `NEW #(...)` pour cohérence avec le reste du code.

---

### HC-028 — Modernisation : `ADD 1 TO` → `+= 1`
**Fichiers :** `zcl_itab_history.clas.abap` (l.416), `zcl_hardcode_shm.clas.abap` (l.260), `zcl_text_utilities.clas.abap` (l.135)
**Recommandation :** Utiliser la syntaxe 7.40 `variable += 1`.

---

### HC-029 — Modernisation : `REFRESH :` → `CLEAR`
**Fichier :** `ztec_hardcode_xpra_transport.prog.abap` ligne 298
**Code :** `REFRESH : lt_source_code, lt_hardcode.`
**Recommandation :** Préférer `CLEAR lt_source_code. CLEAR lt_hardcode.` (ou `lt_source_code = VALUE #( ).`).

---

### HC-030 — `ZCL_ITAB_HISTORY` : Constante `99999999` en dur dans `FACTORY`
**Fichier :** `zcl_itab_history.clas.abap` ligne 122
**Code :** `lv_history_version_max = 99999999.`
**Recommandation :** Utiliser la constante déjà définie `mc_history-version_max` (valeur `999999999`). Attention : les deux valeurs sont différentes (8 vs 9 chiffres) — potentiel bug mineur.

---

> ## Findings complémentaires — Audit ZCL_HARDCODE_MAINTAIN et ZCL_GUI_ALV_TREE_UTIL (2026-03-17)
> Ces deux fichiers n'avaient pu être analysés complètement lors de la session initiale.

---

## 🔴 Critical (suite)

### HC-031 — `ZCL_HARDCODE_MAINTAIN` : Affectation du mauvais champ dans `_PARAMETER_NEW`
**Fichier :** `zcl_hardcode_maintain.clas.abap` ligne 2788
**Code :**
```abap
ls_hardcode_parameter-version_guid = cl_abap_random=>seed( ).
```
**Problème :** Le champ `version_guid` est alimenté avec une graine aléatoire. D'après la logique du contexte (création d'un nouveau paramètre), le champ attendu est `parameter_guid`. Ce bug peut provoquer une liaison incorrecte entre paramètre et version, corrompant silencieusement les données.
**Correction :** Remplacer `version_guid` par `parameter_guid`.

---

### HC-032 — `ZCL_HARDCODE_MAINTAIN` : Affectation en cascade ambiguë dans `_ATTRIBUTE_GET`
**Fichier :** `zcl_hardcode_maintain.clas.abap` ligne 480
**Code :**
```abap
rv_attribute_guid_regroup = ls_hardcode_attribute-attribute_guid = ls_hardcode_attribute-attribute_guid_regroup.
```
**Problème :** Double `=` sans parenthèses — l'interprétation ABAP de cette chaîne d'affectation est non évidente et peut ne pas produire le résultat attendu. L'intention semble être d'affecter `attribute_guid_regroup` à la fois à `rv_attribute_guid_regroup` et à `ls_hardcode_attribute-attribute_guid`.
**Correction :** Séparer en deux instructions explicites :
```abap
ls_hardcode_attribute-attribute_guid = ls_hardcode_attribute-attribute_guid_regroup.
rv_attribute_guid_regroup = ls_hardcode_attribute-attribute_guid_regroup.
```

---

### HC-033 — `ZCL_HARDCODE_MAINTAIN` : `SELECT *` sur table e071 (SAP standard volumineuse)
**Fichier :** `zcl_hardcode_maintain.clas.abap` ligne 3896
**Code :**
```abap
SELECT * FROM e071 WHERE trkorr EQ @rv_trkorr INTO TABLE @lt_e071.
```
**Problème :** `e071` est la table des objets de transport — elle peut contenir des millions de lignes en production. `SELECT *` ramène toutes les colonnes inutilement. La clause `WHERE trkorr = ...` limite l'ensemble mais le volume reste potentiellement élevé.
**Correction :** Sélectionner uniquement les colonnes nécessaires : `SELECT trkorr pgmid object obj_name FROM e071 WHERE trkorr EQ @rv_trkorr INTO TABLE @lt_e071`.

---

### HC-034 — `ZCL_GUI_ALV_TREE_UTIL` : CATCH vide sur erreur de création de structure
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` ligne 820
**Code :**
```abap
CATCH cx_sy_struct_creation cx_sy_table_creation.
  " (aucun handler)
ENDTRY.
```
**Problème :** Si la création dynamique de structure ou de table échoue, l'exception est silencieusement avalée. L'appelant n'a aucun moyen de savoir que l'objet n'a pas été créé, ce qui peut provoquer un dump en aval sur un objet `INITIAL`.
**Correction :** Lever une exception métier ou retourner un code retour d'erreur. Au minimum, logger via `MESSAGE` ou `sy-msgty = 'E'`.

---

## 🟠 Major (suite)

### HC-035 — `ZCL_HARDCODE_MAINTAIN` : Multiples `SELECT *` sur tables de paramétrage
**Fichier :** `zcl_hardcode_maintain.clas.abap` lignes 2504, 2512, 2524, 2584, 2599
**Code :**
```abap
SELECT * FROM ztec_t_hc_vers ...
SELECT * FROM ztec_t_hc_param ... FOR ALL ENTRIES IN ...
SELECT * FROM ztec_t_hc_attr  ... FOR ALL ENTRIES IN ...
SELECT * FROM ztec_t_hc_param WHERE version_guid NOT IN ( SELECT ... )
SELECT * FROM ztec_t_hc_attr  WHERE parameter_guid NOT IN ( SELECT ... )
```
**Problème :** `SELECT *` transfère toutes les colonnes, y compris `mandt` et les colonnes administratives. Coût réseau et mémoire inutile. Les sous-requêtes `NOT IN (SELECT ...)` peuvent aussi être coûteuses.
**Correction :** Lister explicitement les colonnes nécessaires. Remplacer les `NOT IN (SELECT)` par `LEFT OUTER JOIN ... WHERE ... IS NULL` si supporté, ou par une logique ABAP avec tables internes.

---

### HC-036 — `ZCL_HARDCODE_MAINTAIN` : Affectation directe de `sy-mandt` dans les données techniques
**Fichier :** `zcl_hardcode_maintain.clas.abap` lignes 455, 1995–1997, 2772, 3497, 3522, 3546
**Code :**
```abap
ls_hardcode_attribute-mandt = sy-mandt.
```
**Problème :** Le mandant est affecté manuellement depuis `sy-mandt` dans les structures avant `INSERT/MODIFY`. Ce pattern est redondant (le framework SAP gère le mandant lors des opérations DB) et fragile dans un contexte multi-mandant ou RFC cross-client.
**Correction :** Supprimer les affectations manuelles de `mandt`. Si l'initialisation explicite est requise pour des raisons techniques, la documenter et centraliser dans une méthode dédiée.

---

### HC-037 — `ZCL_HARDCODE_MAINTAIN` : `CATCH cx_root` sans logging dans les opérations de sauvegarde
**Fichier :** `zcl_hardcode_maintain.clas.abap` lignes 3604, 3633, 3662
**Code :**
```abap
CATCH cx_root.
  lv_subrc = 4.
ENDTRY.
```
**Problème :** Les exceptions génériques `cx_root` sont capturées et traduites en `lv_subrc = 4` sans aucun logging du message d'erreur. En cas d'échec d'une opération `INSERT/MODIFY/DELETE`, le développeur n'a aucune trace de la cause réelle.
**Correction :** Ajouter `INTO DATA(lx_root)` et logger `lx_root->get_text( )` ou `lx_root->if_message~get_longtext( )`.

---

### HC-038 — `ZCL_GUI_ALV_TREE_UTIL` : Variable `ls_hierarchy` testée après CLEAR
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` ligne 370 (CLEAR à la ligne 265)
**Problème :** `ls_hierarchy` est vidée par `CLEAR ls_hierarchy` à la ligne 265 dans la même méthode. À la ligne 370, une condition `IF NOT ls_hierarchy-node_level_key_fieldname IS INITIAL` teste un champ qui est forcément vide — la branche n'est donc jamais empruntée et la logique conditionnelle est morte.
**Correction :** Remplacer `ls_hierarchy` par `is_hierarchy_lower` (le paramètre import contenant les données de la hiérarchie inférieure) ou restaurer la valeur avant la condition.

---

### HC-039 — `ZCL_GUI_ALV_TREE_UTIL` : Concaténation self-référentielle avec variable non initialisée
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` ligne 1166
**Code :**
```abap
ls_level_node_key-node_text_complete = |{ iv_parent_node_text }{ ls_level_node_key-node_text_complete }|.
```
**Problème :** `ls_level_node_key-node_text_complete` est référencé à droite du template alors qu'il vient d'être créé et n'est pas initialisé. Le résultat est `iv_parent_node_text` concaténé avec une chaîne vide — la partie droite est silencieusement ignorée.
**Correction :** Utiliser le champ source correct, probablement `ls_level_node_key-node_text` :
```abap
ls_level_node_key-node_text_complete = |{ iv_parent_node_text }{ ls_level_node_key-node_text }|.
```

---

## 🟡 Minor (suite)

### HC-040 — `ZCL_HARDCODE_MAINTAIN` : Mauvais préfixe `lr_` pour une table interne
**Fichier :** `zcl_hardcode_maintain.clas.abap` ligne 621
**Code :** `lr_ranges_value` déclaré comme table de ranges
**Problème :** Le préfixe `lr_` désigne une référence d'objet en convention SAP. Une table interne de type ranges doit utiliser `lt_` (ou `lrt_` si ranges).
**Correction :** Renommer en `lt_ranges_value` ou `lrt_ranges_value`.

---

### HC-041 — `ZCL_HARDCODE_MAINTAIN` : Logique DELETE potentiellement incorrecte
**Fichier :** `zcl_hardcode_maintain.clas.abap` lignes 1091–1092
**Problème :** Le DELETE supprime des attributs en utilisant `attribute_guid_regroup` comme critère de comparaison avec lui-même — intention peu claire. Le code pourrait supprimer des attributs qu'il ne faut pas supprimer, ou ne jamais supprimer ceux qu'il faut.
**Correction :** Clarifier la condition et la documenter avec un commentaire explicite sur l'intention métier.

---

### HC-042 — `ZCL_HARDCODE_MAINTAIN` : Confusion de paramètre dans la construction du message d'erreur
**Fichier :** `zcl_hardcode_maintain.clas.abap` lignes 3269–3270
**Problème :** Le paramètre `iv_version_guid` est utilisé comme `iv_msgv2` dans la levée d'exception, alors que le contexte suggère qu'il devrait alimenter `iv_version_guid`. Risque de message d'erreur trompeur.
**Correction :** Vérifier le mapping des paramètres `iv_msgv1` / `iv_msgv2` / `iv_version_guid`.

---

### HC-043 — `ZCL_GUI_ALV_TREE_UTIL` : Séparateur de chemin Windows hardcodé
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` ligne 899
**Code :**
```abap
lv_filename = |{ iv_filepath }\\{ iv_filename }|.
```
**Problème :** Le séparateur `\\` est spécifique Windows. Non portable (bien qu'ABAP soit majoritairement sur Windows) et non maintenable.
**Correction :** Utiliser `cl_gui_frontend_services=>path_separator` ou la constante système appropriée.

---

### HC-044 — `ZCL_GUI_ALV_TREE_UTIL` : En-têtes de méthode obsolètes
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` lignes 162–182, 394–413, 450–470 (et autres)
**Problème :** Les en-têtes contiennent des dates de création, des noms de développeurs et des listes de modifications (pattern SAP classique). Ces informations sont redondantes avec l'historique abapgit et nuisent à la lisibilité.
**Recommandation :** Nettoyer et remplacer par un commentaire court d'intention.

---

## 🔵 Suggestion (suite)

### HC-045 — Modernisation `ZCL_HARDCODE_MAINTAIN` : `MOVE-CORRESPONDING` → `CORRESPONDING #(...)`
**Fichier :** `zcl_hardcode_maintain.clas.abap` lignes 560, 580, 1326, 2431, 2434, 2437, 2447, 2450, 2462, 2802
**Recommandation :** Remplacer les `MOVE-CORRESPONDING ... TO ...` par l'opérateur `CORRESPONDING #(...)` pour cohérence avec le style ABAP 7.40 déjà utilisé dans ce fichier.

---

### HC-046 — Modernisation `ZCL_HARDCODE_MAINTAIN` : `CREATE OBJECT` → `NEW #(...)`
**Fichier :** `zcl_hardcode_maintain.clas.abap` ligne 2115
**Code :** `CREATE OBJECT ro_hardcode_maintain EXPORTING iv_repid = lv_repid_new.`
**Recommandation :** `ro_hardcode_maintain = NEW #( lv_repid_new ).`

---

### HC-047 — Modernisation `ZCL_GUI_ALV_TREE_UTIL` : `CREATE OBJECT` (×3) et `MOVE-CORRESPONDING`
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` lignes 507, 656, 941, 942
**Recommandation :**
- L.507 : `lo_alv_tree_util = NEW zcl_gui_alv_tree_util( co_alv_tree ).`
- L.941 : `lo_excel = NEW zcl_excel( ).`
- L.942 : `lo_excel_writer = NEW zcl_excel_writer_2007( ).`
- L.656 : `<lfs_data> = CORRESPONDING #( is_data ).`

---

### HC-048 — Modernisation `ZCL_GUI_ALV_TREE_UTIL` : `ADD 1 TO` et `CONCATENATE` legacy
**Fichier :** `zcl_gui_alv_tree_util.clas.abap` lignes 235, 254
**Recommandation :**
- L.235 : `lv_node_level += 1.`
- L.254 : `lv_key = |{ lv_key }{ lv_key_value }|.`
