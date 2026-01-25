# Format CRO – Containers ROms pour Amstrad CPC

Ce document décrit le format **CRO** (*Containers ROms*), un format de conteneur destiné à regrouper et décrire des ROMs pour les ordinateurs **Amstrad CPC**, toutes générations confondues.

Le format CRO est basé sur le **Resource Interchange File Format (RIFF)** et en exploite les mécanismes d’extensibilité et de structuration en blocs (*chunks*).

---

## 1. Rappels sur le format RIFF

### 1.1 Principe général

RIFF est un format de conteneur binaire organisé en **chunks** indépendants. Il ne définit pas la sémantique des données qu’il transporte, mais uniquement leur organisation structurelle.

Un fichier RIFF est constitué :

* d’un en-tête global,
* suivi d’une suite de chunks, éventuellement imbriqués.

Les chunks inconnus peuvent être ignorés sans compromettre l’analyse du fichier.

---

### 1.2 En-tête RIFF

Tout fichier RIFF commence par un en-tête de 12 octets :

| Offset | Taille | Description                                                    |
| -----: | -----: | -------------------------------------------------------------- |
|   0x00 |      4 | Identifiant ASCII `"RIFF"`                                     |
|   0x04 |      4 | Taille totale du fichier moins 8 octets (uint32 little-endian) |
|   0x08 |      4 | *Form Type* identifiant le format applicatif                   |

Dans le cas du format CRO, le *Form Type* est `"CRO "`.

---

### 1.3 Structure d’un chunk RIFF

Chaque chunk suit la structure suivante :

| Champ      | Taille | Description                             |
| ---------- | -----: | --------------------------------------- |
| Chunk ID   |      4 | Identifiant ASCII                       |
| Chunk Size |      4 | Taille des données du chunk (uint32 LE) |
| Chunk Data |      N | Données                                 |
| Padding    | 0 ou 1 | Alignement sur taille paire             |

Si `Chunk Size` est impair, un octet de padding est ajouté après les données. Cet octet n’est pas comptabilisé dans `Chunk Size` mais fait partie de la taille globale du fichier.

---

## 2. Identification d’un fichier CRO

Un fichier CRO est un fichier RIFF dont le *Form Type* est :

```
"CRO "
```

Il décrit une organisation hiérarchique de ROMs destinée aux systèmes CPC.

---

## 3. Vue d’ensemble de la structure CRO

La structure logique d’un fichier CRO est la suivante :

```
RIFF "CRO "
 ├── GRRO (groupe de ROMs)
 │    ├── Descripteurs du groupe
 │    └── ROM (une ou plusieurs)
 │         ├── Descripteurs de la ROM
 │         └── Données binaires
 └── GRRO …
```

Un fichier CRO contient un ou plusieurs **GRRO**, chacun regroupant un ensemble cohérent de ROMs.

---

## 4. Chunk GRRO – Regroupement de ROMs

Le chunk **GRRO** (*GRoupement de ROms*) décrit un groupe logique de ROMs.

### 4.1 Sous-chunks du GRRO

Un chunk GRRO contient les sous-chunks suivants, dans un ordre libre :

| Chunk ID | Rôle                         |
| -------- | ---------------------------- |
| `GNUM`   | Numéro unique du GRRO        |
| `GLBL`   | Libellé descriptif du groupe |
| `ROM `   | Description d’une ROM        |

Un GRRO valide contient exactement **un** chunk `GNUM`, **un** chunk `GLBL`, et **un ou plusieurs** chunks `ROM `.

### 4.2 Structure binaire du chunk GRRO

Le chunk GRRO suit la structure RIFF standard :

| Offset | Taille | Description                                       |
| -----: | -----: | ------------------------------------------------- |
|  +0x00 |      4 | Identifiant ASCII `"GRRO"`                        |
|  +0x04 |      4 | Taille du chunk (uint32 LE)                       |
|  +0x08 |      N | Données du chunk (sous-chunks GNUM + GLBL + ROMs) |

---

## 5. Chunk ROM – Description d’une ROM

Chaque ROM est décrite individuellement au sein d’un GRRO.

### 5.1 Sous-chunks du chunk ROM

Un chunk ROM contient les sous-chunks suivants, dans un ordre libre :

| Chunk ID | Rôle                        |
| -------- | --------------------------- |
| `RID `   | Identifiant de la ROM       |
| `RTYP`   | Type logique de la ROM      |
| `RLOG`   | Numéro logique de sélection |
| `RPHY`   | Numéro physique de la ROM   |
| `RDT `   | Données binaires de la ROM  |

Tous les sous-chunks ci-dessus sont obligatoires.

### 5.2 Structure binaire des sous-chunks ROM

#### `RID ` – Identifiant de la ROM

| Offset | Taille | Description                               |
| -----: | -----: | ----------------------------------------- |
|  +0x00 |      N | Chaîne ASCII (non terminée), taille libre |

#### `RTYP` – Type logique de la ROM

| Offset | Taille | Description                           |
| -----: | -----: | ------------------------------------- |
|  +0x00 |      4 | Constante de type logique (uint32 LE) |

Valeurs définies :

* `0x00000000` : ROM_LOW
* `0x00000001` : ROM_HIGH
* `0x00000002` : ROM_BANKABLE
* `0x00000003` : ROM_MF2

#### `RLOG` – Numéro logique

| Offset | Taille | Description                                                                 |
| -----: | -----: | --------------------------------------------------------------------------- |
|  +0x00 |      1 | Numéro logique de sélection (peut différer du numéro physique pour CPR/XPR) |

#### `RPHY` – Numéro physique

| Offset | Taille | Description                                            |
| -----: | -----: | ------------------------------------------------------ |
|  +0x00 |      1 | Numéro physique de la ROM (correspond au chunk `cbXX`) |

#### `RDT ` – Données binaires de la ROM

| Offset | Taille | Description            |
| -----: | -----: | ---------------------- |
|  +0x00 |      N | Contenu brut de la ROM |

---

## 6. Neutralité matérielle du format CRO

Le format CRO décrit :

* l’organisation des ROMs,
* leurs propriétés intrinsèques,
* leurs regroupements logiques.

Il ne décrit pas :

* les ports matériels,
* les registres de sélection,
* les mécanismes d’activation ou de mapping.

L’interprétation de ces informations relève de l’émulation ou du matériel réel, selon les capacités effectivement disponibles.

Pour CPR/XPR, certaines ROMs physiques ont des numéros logiques différents (exemple : `cb03` → logique 7) ; cette distinction est conservée dans `RLOG`.

---

## 7. Extensibilité

Grâce à l’utilisation de RIFF :

* de nouveaux chunks peuvent être ajoutés sans rompre la compatibilité,
* des propriétés supplémentaires peuvent être introduites ultérieurement,
* les implémentations peuvent ignorer les chunks non reconnus.

Le format CRO est ainsi conçu pour être durable, extensible et fidèle aux architectures CPC existantes ou futures.
