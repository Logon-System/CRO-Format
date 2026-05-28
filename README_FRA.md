# Format CRO – Conteneur de ROMs pour Amstrad CPC
![Image Presentation CRO format](./images/logocro.png)
Ce document décrit le format **CRO** (*Conteneur de ROms*), un format destiné à regrouper et décrire les ROMs pour les ordinateurs **Amstrad CPC**, toutes générations confondues.

Le format CRO repose sur le **Resource Interchange File Format (RIFF)** et utilise sa structure par chunks pour organiser les données.

---

## 1. Vue générale du format RIFF

### 1.1 Principe général

RIFF est un format binaire organisé en **chunks indépendants**. Il ne définit pas la signification des données, seulement l’organisation structurelle.

Un fichier RIFF est constitué de :

* un en-tête global,
* suivi d’une séquence de chunks, pouvant être imbriqués.

Les chunks inconnus peuvent être ignorés sans casser la lecture.

---

### 1.2 En-tête RIFF

Tous les fichiers RIFF commencent par un en-tête de 12 octets :

| Offset | Taille | Description                                   |
| ------ | ------ | --------------------------------------------- |
| 0x00   | 4      | Identifiant ASCII `"RIFF"`                     |
| 0x04   | 4      | Taille totale du fichier moins 8 octets (uint32 LE) |
| 0x08   | 4      | *Form Type* identifiant le format applicatif  |

Pour CRO, le *Form Type* est `"CRO "`.

---

### 1.3 Structure des chunks

Chaque chunk est structuré ainsi :

| Champ       | Taille | Description                     |
| ----------- | ------ | --------------------------------|
| Chunk ID    | 4      | Identifiant ASCII                |
| Chunk Size  | 4      | Taille des données (uint32 LE)  |
| Chunk Data  | N      | Données                          |
| Padding     | 0 ou 1 | Alignement à un octet pair       |

Si la taille est impaire, un octet de padding est ajouté **après les données**.

---
### 1.4 Vue globale

![RIFF Vue globale](./images/RIFF.gif)

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
RIFF
└─ CRO␠
├─ GRRO
│ ├─ GLBL
│ ├─ GNUM
│ ├─ GMSK
│ ├─ ROM␠
│ │ ├─ RID␠
│ │ ├─ RTYP
│ │ ├─ RLOG
│ │ ├─ RPHY
│ │ └─ RDT␠
│ └─ ...
└─ ...
Un fichier CRO contient un ou plusieurs **GRRO**, chacun regroupant un ensemble cohérent de ROMs.
```

## 4. Chunk GRRO – Groupe de ROMs

### 4.1 Sous-chunks

| Chunk ID | Rôle |
|--------|------|
| `GNUM` | Numéro unique du GRRO |
| `GLBL` | Libellé descriptif du groupe |
| `GMSK` | Masque d’adressage |
| `ROM ` | Description d’une ROM |

Un GRRO valide doit contenir **un GNUM, un GLBL, un GMSK et une ou plusieurs ROM**.

### 4.2 Structure binaire

| Offset | Taille | Description |
|------|------|------------|
| 0x00 | 4 | Identifiant `"GRRO"` |
| 0x04 | 4 | Taille du chunk (uint32 LE) |
| 0x08 | N | Données (GNUM + GLBL + GMSK + ROMs) |

---
### 4.3 GNUM – Numéro de groupe

| Offset | Taille | Description |
|------|------|------------|
| 0x00 | 4 | Numéro unique du groupe (uint32 LE, commence à 0) |

---

### 4.4 GLBL – Libellé du groupe

| Offset | Taille | Description |
|------|------|------------|
| 0x00 | N | Chaîne ASCII descriptive (non terminée) |

---

### 4.5 GMSK – Masque d’adressage

| Offset | Taille | Description |
|------|------|------------|
| 0x00 | 4 | Masque d’adressage (uint32 LE) |

Le chunk `GMSK` définit un **masque binaire appliqué à l’adresse de la ROM** :
adresse_effective = adresse & GMSK
Ce mécanisme permet d’émuler des EEPROM de taille inférieure à l’espace d’adressage réel
(par exemple l’adressage 19 bits des CPC PLUS), en provoquant une redondance automatique
des ROMs lorsque l’adresse dépasse la capacité réelle du support.

---

## 5. Chunk ROM – Description d’une ROM

Chaque ROM est définie dans un GRRO.

### 5.1 Sous-chunks

| Chunk ID | Rôle                             |
| -------- | -------------------------------- |
| `RID `   | Identifiant de la ROM            |
| `RTYP`   | Type logique de la ROM           |
| `RLOG`   | Numéro logique                   |
| `RPHY`   | Numéro physique                  |
| `RDT `   | Contenu binaire                  |

Tous les sous-chunks sont obligatoires.

### 5.2 Structure binaire

#### `RID ` – Identifiant

| Offset | Taille | Description                  |
| ------ | ------ | ---------------------------- |
| 0x00   | N      | Chaîne ASCII, longueur variable (non terminée) |

#### `RTYP` – Type logique

| Offset | Taille | Description                  |
| ------ | ------ | ---------------------------- |
| 0x00   | 4      | Constante de type (uint32 LE) |

Valeurs définies :

* `0x00000000` : ROM_LOW  
* `0x00000001` : ROM_HIGH  
* `0x00000002` : ROM_BANKABLE  
* `0x00000003` : ROM_MF2

#### `RLOG` – Numéro logique

| Offset | Taille | Description                  |
| ------ | ------ | ---------------------------- |
| 0x00   | 4      | Numéro logique de sélection (uint32 LE, 0–255) |

#### `RPHY` – Numéro physique

| Offset | Taille | Description                  |
| ------ | ------ | ---------------------------- |
| 0x00   | 4      | Numéro physique (uint32 LE)  |

#### `RDT ` – Données binaires

| Offset | Taille | Description                  |
| ------ | ------ | ---------------------------- |
| 0x00   | N      | Contenu brut de la ROM       |

---

## 6. Neutralité matérielle

Le format CRO décrit uniquement :

* l’organisation des ROMs
* leurs propriétés intrinsèques
* leur regroupement logique

Il ne décrit **pas** :

* les ports matériels
* les registres de sélection
* les mécanismes d’activation ou mapping

L’interprétation est laissée à l’émulation ou au hardware réel.

---

## 7. Extensibilité

Grâce à RIFF :

* de nouveaux chunks peuvent être ajoutés sans casser la compatibilité
* de nouvelles propriétés peuvent être introduites
* les implémentations peuvent ignorer les chunks non reconnus

CRO est donc durable, extensible et compatible avec toutes les architectures CPC.
