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
 +-- GRRO (groupe de ROMs)
 ¦    +-- Descripteurs du groupe
 ¦    +-- ROM (une ou plusieurs)
 ¦         +-- Descripteurs de la ROM
 ¦         +-- Données binaires
 +-- GRRO …
```

Un fichier CRO contient un ou plusieurs **GRRO**, chacun regroupant un ensemble cohérent de ROMs.

---

## 4. Chunk GRRO – Regroupement de ROMs

Le chunk **GRRO** est un chunk RIFF contenant des sous-chunks. Il définit un regroupement logique de ROMs et sert de conteneur aux chunks ROM qu’il contient.

### 4.0 Structure binaire du chunk GRRO

Le chunk GRRO suit la structure RIFF standard :

| Offset | Taille | Description                 |
| -----: | -----: | --------------------------- |
|  +0x00 |      4 | Identifiant ASCII `"GRRO"`  |
|  +0x04 |      4 | Taille du chunk (uint32 LE) |
|  +0x08 |      N | Données du chunk            |

Les données du chunk GRRO sont constituées exclusivement de **sous-chunks RIFF**, décrits ci-dessous.

Le chunk **GRRO** (*GRoupement de ROms*) décrit un groupe logique de ROMs.

### 4.1 Rôle du GRRO

Un GRRO représente un ensemble de ROMs pouvant être :

* sélectionnées conjointement,
* activées ou désactivées comme un tout,
* redirigées vers un autre groupe par un mécanisme matériel ou logiciel.

Ce concept permet de représenter fidèlement les cartes ROM et cartouches capables de commuter des ensembles complets de ROMs.

---

### 4.1 Sous-chunks du GRRO

Un chunk GRRO contient les sous-chunks suivants, dans un ordre libre :

| Chunk ID | Rôle                         |
| -------- | ---------------------------- |
| `GNUM`   | Numéro unique du GRRO        |
| `GLBL`   | Libellé descriptif du groupe |
| `ROM `   | Description d’une ROM        |

Un GRRO valide contient exactement **un** chunk `GNUM`, **un** chunk `GLBL`, et **un ou plusieurs** chunks `ROM `.

Le chunk GRRO contient :

* un en-tête RIFF standard,
* des champs descriptifs propres au groupe,
* une suite de chunks ROM.

#### Champs descriptifs du GRRO

| Champ            | Description                              |
| ---------------- | ---------------------------------------- |
| Numéro de groupe | Identifiant unique du GRRO, débutant à 0 |
| Libellé          | Chaîne descriptive du groupe             |

Le numéro de groupe permet d’identifier un GRRO dans les mécanismes de sélection externes.

---

## 5. Chunk ROM – Description d’une ROM

Le chunk **ROM ** décrit une ROM individuelle et contient à la fois ses propriétés descriptives et ses données binaires.

### 5.0 Structure binaire du chunk ROM

Le chunk ROM suit la structure RIFF standard :

| Offset | Taille | Description                 |
| -----: | -----: | --------------------------- |
|  +0x00 |      4 | Identifiant ASCII `"ROM "`  |
|  +0x04 |      4 | Taille du chunk (uint32 LE) |
|  +0x08 |      N | Données du chunk            |

Les données du chunk ROM sont constituées de sous-chunks RIFF, décrits ci-dessous.

Chaque ROM est décrite individuellement au sein d’un GRRO.

### 5.1 Rôle du chunk ROM

Un chunk ROM représente une ROM physique ou logique, telle qu’elle existe sur le matériel CPC ou sur une carte d’extension.

Il associe :

* des propriétés descriptives,
* le contenu binaire de la ROM.

---

### 5.1 Sous-chunks du chunk ROM

Un chunk ROM contient les sous-chunks suivants, dans un ordre libre :

| Chunk ID | Rôle                        |
| -------- | --------------------------- |
| `RID `   | Identifiant de la ROM       |
| `RSIZ`   | Taille réelle de la ROM     |
| `RTYP`   | Type logique de la ROM      |
| `RLOG`   | Numéro logique de sélection |
| `RPHY`   | Numéro physique de la ROM   |
| `RDT `   | Données binaires de la ROM  |

Tous les sous-chunks ci-dessus sont obligatoires.

Les champs suivants décrivent une ROM :

| Champ           | Description                                      |
| --------------- | ------------------------------------------------ |
| Identifiant     | Nom ou identifiant symbolique                    |
| Taille          | Taille réelle de la ROM en octets                |
| Type logique    | Nature de la ROM (voir section suivante)         |
| Numéro logique  | Numéro auquel la ROM répond lors de la sélection |
| Numéro physique | Position physique réelle de la ROM               |

Ces champs décrivent les caractéristiques intrinsèques de la ROM, indépendamment de tout mécanisme matériel particulier.

---

### 5.2 Structure binaire des sous-chunks ROM

#### `RID ` – Identifiant de la ROM

| Offset | Taille | Description                               |
| -----: | -----: | ----------------------------------------- |
|  +0x00 |      N | Chaîne ASCII (non terminée), taille libre |

---

#### `RSIZ` – Taille de la ROM

| Offset | Taille | Description                            |
| -----: | -----: | -------------------------------------- |
|  +0x00 |      4 | Taille de la ROM en octets (uint32 LE) |

---

#### `RTYP` – Type logique de la ROM

| Offset | Taille | Description                           |
| -----: | -----: | ------------------------------------- |
|  +0x00 |      4 | Constante de type logique (uint32 LE) |

Valeurs définies :

* `0x00000000` : ROM_LOW
* `0x00000001` : ROM_HIGH
* `0x00000002` : ROM_BANKABLE

---

#### `RLOG` – Numéro logique

| Offset | Taille | Description                 |
| -----: | -----: | --------------------------- |
|  +0x00 |      1 | Numéro logique de sélection |

---

#### `RPHY` – Numéro physique

| Offset | Taille | Description               |
| -----: | -----: | ------------------------- |
|  +0x00 |      1 | Numéro physique de la ROM |

---

#### `RDT ` – Données binaires de la ROM

| Offset | Taille | Description            |
| -----: | -----: | ---------------------- |
|  +0x00 |      N | Contenu brut de la ROM |

Le type logique décrit la nature de la ROM dans l’architecture CPC, sans référence à un modèle particulier ni à un état de mapping.

Les types logiques définis sont :

| Constante    | Signification                                                                  |
| ------------ | ------------------------------------------------------------------------------ |
| ROM_LOW      | ROM dédiée à l’espace bas (`&0000`)                                            |
| ROM_HIGH     | ROM accessible uniquement dans l’espace haut (`&C000`)                         |
| ROM_BANKABLE | ROM appartenant à l’ensemble bancable, susceptible d’être mappée dynamiquement |

Un seul type logique est associé à chaque ROM.

---

### 5.3 Données binaires de la ROM

Le sous-chunk `RDT ` contient le contenu brut de la ROM. Sa taille doit correspondre exactement à la valeur indiquée dans le sous-chunk `RSIZ`.

Les données binaires correspondent au contenu brut de la ROM.

* Aucune hypothèse n’est faite sur leur organisation interne.
* La taille est donnée explicitement par le champ `Taille`.

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

---

## 7. Extensibilité

Grâce à l’utilisation de RIFF :

* de nouveaux chunks peuvent être ajoutés sans rompre la compatibilité,
* des propriétés supplémentaires peuvent être introduites ultérieurement,
* les implémentations peuvent ignorer les chunks non reconnus.

Le format CRO est ainsi conçu pour être durable, extensible et fidèle aux architectures CPC existantes ou futures.
