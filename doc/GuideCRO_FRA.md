# Table des Matières

1. [Introduction](#introduction)
2. [La gamme Amstrad Plus et le format CPR](#la-gamme-amstrad-plus-et-le-format-cpr)
3. [Limitations du format CPR](#limitations-du-format-cpr)
4. [Nouvelles cartes et gestion moderne des ROMs](#nouvelles-cartes-et-gestion-moderne-des-roms)
    - [Carte C4CPC](#carte-c4cpc)
    - [Carte PICO GX](#carte-pico-gx)
5. [En pratique : gestion des fichiers CRO](#en-pratique-gestion-des-fichiers-cro)
    - [Masque d'adresse et compatibilité EEPROM](#masque-dadresse-et-compatibilite-eeprom)
    - [Copie des ROMs dans la RAM](#copie-des-roms-dans-la-ram)
6. [Traitement d'une adresse par le CPC](#traitement-dune-adresse-par-le-cpc)
7. [Conclusion](#conclusion)

---
# Introduction

Il existe très peu de fichiers conteneurs pour les roms sur CPC.

Sur les CPC d'ancienne génération, la machine contient 2 ou 3 rom selon le modèle. Il est néanmoins possible d'ajouter des roms complémentaires.  

Ces roms supplémentaires sont souvent créées pour supporter le hardware de cartes d'extension. C'est notamment le cas lorsque ces cartes contiennent des mémoires de masse car la rom disque native n'est pas capable de gérer cette mémoire. C'est aussi le cas de quelques outils utilitaires nécessitant une faible empreinte mémoire.

Le firmware du CPC est prévu pour coexister de manière standard des roms additionnelles lorsque la machine démarre, en limitant toutefois le nombre de roms initialisées au niveau software. Le système essaie d'initialiser jusqu'à 15 roms sur 6128, alors que la machine peut techniquement adresser 256 roms haute différentes.

Des cartes ont été conçues pour permettre de regrouper des roms de manière plus ou moins pratique. Mais leur mise à jour au sein de ces cartes a toujours été unitaire et fastidieuse, sans qu'un format conteneur soit adopté. Cela peut s'expliquer historiquement par la faible capacité des disquettes pour supporter de gros fichiers et à la capacité ram du Cpc. À l'époque les fichiers roms étaient encore stockés sur ces supports et transitaient par la ram.  

Les auteurs des premiers émulateurs qui ont émulé les CPC ont mis au point un format DSK pour émuler le contenu de disquettes. Mais au niveau des roms, rien n'a été réalisé. Les roms sont restées unitaires, et l'affectation de ces roms au sein de l'émulation a souvent lieu dans des menus avec des dizaines d'emplacements dont l'usage se révèle souvent rébarbatif lorsqu'il faut paramétrer plus de 3 ROMs. Cela nécessite des profils de paramétrage (lorsque c'est prévu) pour pouvoir basculer entre des configurations différentes.

---

# La gamme Amstrad Plus et le format CPR

Avec la création de la gamme des Amstrad Plus, Amstrad a créé un système propriétaire de cartouches pouvant contenir jusqu'à 32 roms de 16k chacune pour un total de 512k de roms. Il s'agissait ici d'une limitation hardware du nombre de rom gérables par le CPC d'origine (256 via le port d'extension contre 32 via le port cartouche des Plus/GX).  

Très peu de roms commerciales sont sorties sur la machine (car la GX4000 a été un flop commercial). Elles ont néanmoins été "dumpées" et un premier format d'encapsulation (CPR) des roms a été créé pour pouvoir être utilisé dans les premiers émulateurs. Une des premières cartes populaire capable de se substituer à une cartouche officielle en hackant le système de protection était la carte C4CPC. Cette carte permet de sélectionner d'un coup un groupe de rom encapsulés dans un CPR.  

Le paramétrage de CPR dans les premiers émulateurs (comme Winape) est indigeste. Il est venu "coexister" avec l'ancien système de sélection des roms, engendrant quelques bugs de mapping (le no de rom est par exemple "andé" avec 31 avant d'être comparé à 7, ce qui permet de sélectionner la rom 7 sur &07,&27,..). Si l'émulateur peut "charger" un fichier CPR, l'utilisateur est toujours tenu d'utiliser le menu de sélection de rom pour aller paramétrer la première "rom" de la cartouche manuellement. Les émulateurs plus récents acceptent désormais qu'un fichier CPR soit directement chargé en drag'n'drop et soit automatiquement chargé et configuré par l'émulateur, ces derniers dissocient mieux les deux systèmes de gestion (ancienne gamme, Plus).  

---
# Limitations du format CPR

Le format CPR a été construit à partir d'une méthode standard appelée "RIFF". 
Il est cependant simpliste car les roms ne possèdent aucune propriété (taille, type). 
Les roms font toutes 16k et sont encapsulées dans une limite de 512k.  

Les machines PLUS sont rétro compatibles avec l'ancienne gamme, sur laquelle il existe une notion de rom logique. C'est notamment le cas avec la rom disque (AMSDOS) qui possède le numéro logique 7, bien que la machine ne dispose que de 3 roms au maximum (6128 et 664 pour les anciennes génération). Lorsqu'un programme sur l'ancienne gamme sélectionne la rom 7, cette dernière sait électroniquement qu'elle est sollicitée et répond présent. Son "ordre physique" importe peu et n'a que peu de sens.  

Afin de conserver cette compatibilité avec l'ancienne gamme, les ingénieurs Amstrad, qui avaient déjà sacrifié le nombre de roms possible de 256 à 32 pour la gamme du Plus, se sont dit qu'ils pouvaient sacrifier la capacité d'accès du nombre de roms classique de l'ancien système émulé pour les cartouches de 256 à 128. Via l'utilisation du bit 7 du numéro de rom (sur le port de sélection &DFFF), cela permet de sélectionner une rom par transposition logique (bit 7=0), ou de sélectionner une rom par son numéro physique (bit 7=1). Les ingénieurs ont peut-être supposé que personne n'utiliserait plus de 128 Roms sur un CPC. Ce qui était vrai dans les années 90 ne l'est sans doute plus aujourd'hui.

Les roms d'une cartouche "Plus" étant "stockées" au sein d'une même mémoire limitée à 512k, une table de transposition logique existe pour assurer la compatibilité avec les programmes de l'ancienne gamme accédant à la rom 7. En l'occurrence, pour la rom logique 7, la rom physique correspondante est la numéro 3 dans une cartouche. Autrement dit, sur la gamme des Plus, faire `OUT &DF00,7` revient à faire `OUT &DF00,&80+3`. Si la rom logique n'est pas 7, alors la rom physique est la 1, pour reproduire le comportement de l'ancienne génération, qui sélectionne la 1ère rom haute quelque soit la valeur envoyée sur le port de sélection (à l'exception de la 7). La rom 0 "physique" dans la cartouche est réservée à la rom "basse" du CPC d'ancienne génération. Sur ce dernier point, la gamme des PLUS permet aussi de sélectionner une des 8 premières rom physique de la cartouche comme rom basse (mais également de fixer la base de cette rom ailleurs qu'en &0000 (&4000 ou &8000 via un registre de l'ASIC appelé RMR2)).  

Lorsque Amstrad a sorti en France une cartouche sans le jeu "Burnin' Rubber" à cause de problèmes de compatibilité (lié au bidouillage de la rom disque par Amstrad pour y caser le menu de sélection entre le jeu et le basic), ils ont créé une cartouche contenant 4 roms au lieu de 3, puisque la rom 7 est physiquement la 4ème et que la technologie de l'époque orientait le choix technique vers une eeprom linéaire. Ainsi, dans cette cartouche, ainsi que dans le CPR qui en a été extrait, on trouve en position 3 (numéro 2) une rom qui ne sert absolument à rien (vestige du jeu), sinon d'occuper l'espace.  

Le CPR n'est également pas adapté pour pouvoir encapsuler des ROMS pour les CPCs d'ancienne génération.

---

# Nouvelles cartes et gestion moderne des ROMs

De nouvelles cartes permettent aujourd'hui de dépasser les limitations du nombre de roms accessibles par le hardware d'origine.

Ainsi, sur la gamme des PLUS, la carte C4CPC est "multi-cartouches" et contient plusieurs fichiers CPR. Cette carte est capable de gérer le filesystem contemporain d'une SDCARD contenant plusieurs cartouches, d'en sélectionner une, et de traiter son contenu pour en extraire le contenu dans une "ram" simulant une "cartouche" complète (composée de plusieurs "segments-roms" de 16 k).

Cependant, le hardware "CPC PLUS" n'a aucun moyen de communiquer avec la cartouche car il n'existe pas d'interface d'E/S standardisée dédiée et aucune ligne du port cartouche ne permet de gérer des écritures depuis le Z80A. L'ASIC des PLUS génère 19 bits d'adresse : 14 bits d'adresse pour la page de 16k (A0..A13) et 5 bits de sélection de rom (&DF80+0 à &DF80+&1F).

Pour pouvoir communiquer avec une carte simulant une ROM, il reste donc une méthode, qui consiste à "surveiller" en permanence la lecture d'adresses précises. La carte détecte une combinaison de lectures successives à des adresses précises afin d'éviter qu'un programme active par inadvertance une fonction étendue de la carte, et que cela ne condamne l'usage de certaines adresses.  

Il est d'ailleurs fortement déconseillé de créer un système trop simple qui condamnerait l'usage de certaines adresses ou qui risquerait de provoquer des sollicitations non voulues de la carte.

---
## Carte C4CPC

La méthode employée est la suivante : un numéro de commande est déterminé grâce au poids faible de l'adresse de lecture de deux adresses 19 bits :  
#57FFx (rom #15, #3FFx) (10101 1111111111xxxxxxxx)
#2BFFx (rom #0A, #3FFx) (01010 1111111111xxxxxxxx)



Pour activer une commande (le no de bit de xxxxxxxx étant la commande), il faut lire les deux adresses indiquées successivement, et ceci 2 fois de suite :  

1. sélectionner la rom #15 (`OUT &DF00,&80+&15`), lire &FFFx (`ld a,(hl)`)  
2. sélectionner la rom #0a (`OUT &DF00,&80+&0a`), lire &FFFx (`ld a,(hl)`)  
3. sélectionner la rom #15 (`OUT &DF00,&80+&15`), lire &FFFx (`ld a,(hl)`)  
4. sélectionner la rom #1a (`OUT &DF00,&80+&0a`), lire &FFFx (`ld a,(hl)`)

La carte gère des valeurs de données ou de status pour les synchronisations à partir de l'adresse la plus haute (rom 31). Cela est notamment nécessaire pour transférer un fichier cartouche. Un registre de status est disponible sur l'adresse `#7FFFE` et un registre de données sur `#7FFFF`. Par exemple, une attente de synchronisation après la commande #40 est d'attendre que le status devienne différent de &C7.  

Note : ces informations sont sujettes à caution car elles résultent d'une analyse de reverse engineering à laquelle je me suis livré sur CprSelect V1.2, étant donné qu'aucune documentation officielle n'a été produite à ma connaissance sur le fonctionnement interne de cette carte.

---

## Carte PICO GX

Une carte actuellement en cours de développement, la PICO GX, utilise la méthode suivante :  

Une séquence de lectures de deux adresses 19 bits met la carte à l'écoute d'un numéro de commande et d'un paramètre :  
#0133C (rom 0, #133C) (00000 0001001100111100)
#025C4 (rom 0, #25C4) (00000 0010010111000100)


Pour activer l'écoute de la carte, il faut lire une fois ces deux adresses dans l'ordre. Puis le numéro de commande est déterminé par l'adresse de lecture. La commande "run" est demandée en lisant l'adresse #0002. Le paramètre de la commande qui vient après est donné en lisant l'adresse dont la valeur correspond à ce paramètre. Tout comme pour la C4CPC, un registre de status est présent en &3FFF (la carte travaille lorsque <> 0).  

(*) Je ne sais pas à cette heure si les 5 bits de poids faible sont pris en compte. Le numéro de commande est déterminé grâce au poids faible de l'adresse de lecture de deux adresses 19 bits.

---

# En pratique : gestion des fichiers CRO

Pour qu'un système hardware ou un émulateur traite correctement un fichier CRO, il faut tenir compte que ce n'est plus l'ordre physique des roms dans le fichier qui détermine leur place "réelle" dans la cartouche. Par le passé, et pour reprendre l'exemple des 4 roms de la cartouche "light" sans le jeu Burnin'rubber, la réalité matérielle imposait qu'une 3ème rom existe avant le 4ème (numéro 3 à partir de 0) dans une eeprom de 64k. Avec les cartes actuelles, cette logique peut changer.

Pour des raisons de rapidité d'accès, les "rom" sont simulées via une "ram". À cause des temps de transfert, il n'est pas envisageable de copier dans cette ram les données d'une carte mémoire pour pouvoir gérer cette transposition d'adressage très rapidement. C'est encore plus vrai si on commence à évoquer des cartouches excédant la limite des 512k et/ou qui répartissent cet espace en groupe de roms plus petits. En effet l'intérêt de pouvoir créer 4 groupes de 8 roms au lieu d'un seul groupe de 32 roms réside dans la capacité de l'ASIC des PLUS de permettre un adressage plus large des seules 8 premières roms de la cartouche (4 plages de 16k de l'espace adressable par le Z80A : &0000,&4000,&8000 (rmr2+rmr) ou &C000 (&df8x+rmr)).

On considère que la base de la ram disponible dans une carte (ou dans une émulation) est modifiable avec une mécanique de commande similaire à celles décrites pour les 2 cartes évoquées précédemment. En considérant qu'une commande fixe, permet de modifier la base de l'adressage, on peut ainsi "déplacer" une fenêtre de 512k dans un espace mémoire plus grand, un peu à la manière de la segmentation des premiers processeurs Intel. On va appeler cette commande **GROUPSELECT** et son paramètre le numéro de groupe (voir chunk GRRO/GNUM).  

Si par exemple la base est fixée est &1FFFF, l'adresse effective de début des données dans la ram de la carte n'est plus 0, mais &1FFFF (16k x 8) lorsque le CPC accèdera aux roms 0 à 7.

La lecture d'un fichier CRO permet de gérer ce type d'organisation au sein d'une carte, grâce à une hiérarchie de groupement de plusieurs roms (GRRO). Un groupe est caractérisé par un libellé (chunk optionnel GLBL), un numéro d'index unique numéroté à partir de 0 (chunk optionnel GNUM), et un masque d'adresse (chunk optionnel GMSK).

Le numéro d'index (GNUM) représente l'index d'un groupe de rom. Il est préférable qu'il soit défini, mais si ce n'est pas le cas, alors le programme de traitement peut numéroter les GRRO dans l'ordre ou il les trouve. C'est ce numéro qui sera envoyé par le CPC à la carte avec la commande GROUPSELECT évoquée précédemment. Autrement dit, la carte a besoin de connaitre, pour un groupe GNUM, l'adresse de début des données qui sera la base d'accès à laquelle sera ajoutée l'adresse 19 bits fournie par le CPC PLUS.

---

## Masque d'adresse et compatibilité EEPROM

Il est également possible de définir un masque d'adresse pour un groupe de ROMs. Ce masque permet d'assurer une rétro-compatibilité avec le traitement implicite des Eeprom par le CPC. En effet la taille d'une Eeprom dépend du nombre de bits d'adresse qu'elle gère. Sur CPC PLUS et ses cartouches d'origine, on pourrait avoir des roms allant de 16k (14 bits) à 512k (19 bits). Si un CPC PLUS essaie d'accéder a une adresse plus grande que celle que permet la capacité d'une Eeprom, les bits d'adresse excédentaires ne sont pas pris en compte (and implicite). La conséquence est un cyclage des roms. Ainsi avec une cartouche "eeprom" de 128k (8 segments de 16k), sélectionner la rom 8 ou 16 revient à sélectionner la rom 0. La définition d'un masque appliqué à l'adresse permet de gérer cette redondance si nécessaire.

---

## Copie des ROMs dans la RAM

Lorsque la carte lit un fichier CRO, elle copie dans la ram disponible, à partir de 0, les ROMS disponibles dans chaque groupe en tenant compte de leur numéro physique.

La solution la plus rapide, pour éviter une conversion d'adresse (et permettre de stocker en RAM uniquement les seules ROMs physiques), consiste à laisser des "trous" de 16k dans la RAM pour les ROMs qui n'auraient pas été définies dans un groupe.  

**Exemple : Groupe 0** (ROMs physiques 0, 1, 4, 6)  
| Adresse RAM | ROM | Groupe |
|-------------|-----|-------|
| &00000      | 0   | 0     |
| &04000      | 1   | 0     |
| &08000      | vide| 0     |
| &0C000      | vide| 0     |
| &10000      | 4   | 0     |
| &14000      | vide| 0     |
| &18000      | 6   | 0     |

**Exemple : Groupe 1** (ROMs physiques 0, 2, 4)  
| Adresse RAM | ROM | Groupe |
|-------------|-----|-------|
| &1C000      | 0   | 1     |
| &20000      | vide| 1     |
| &24000      | 2   | 1     |
| &28000      | vide| 1     |
| &2C000      | 4   | 1     |

La base du groupe 1 de ROMs, sélectionné par une commande **GROUPSELECT 1** à la carte, sera donc &1C000. C'est à cette base qu'est ajouté l'adresse 19 bits fournie par le CPC. Et ainsi de suite.

Ce serait plus coûteux qu'une simple addition de l'adresse de base avec l'adresse 19 bits, mais il est aussi possible d'économiser la place occupée par les ROMs non présentes en réalisant une translation des adresses de ROM dans un groupe en fonction de leur présence effective.

**Exemple : Groupe 0** (ROMs physiques 0, 1, 4, 6) avec translation d'adresses  
| Adresse 19 bits ASIC | Adresse RAM Carte | ROM | Groupe |
|---------------------|-----------------|-----|-------|
| &00000..&03FFF      | &00000..&03FFF  | 0   | 0     |
| &04000..&07FFF      | &04000..&07FFF  | 1   | 0     |
| &08000..&0BFFF      | traitement norom | –   | 0     |
| &0C000..&0FFFF      | traitement norom | –   | 0     |
| &10000..&13FFF      | &08000..&0BFFF  | 4   | 0     |
| &14000..&17FFF      | traitement norom | –   | 0     |
| &18000..&1BFFF      | &0C000..&0FFFF  | 6   | 0     |

Une ROM "vide" ou "norom" renvoie toujours 0 sur toutes les adresses lues.

---

# Traitement d'une adresse par le CPC

Le CPC fournit une adresse 19 bits via son port cartouche (A0..A13, AC14..AC18), qu'on appellera `CpcBusAddress`. Dans une structure issue du fichier CRO, on dispose, pour chaque groupe, de :  
- son numéro de groupe  
- son adresse de base  
- son masque (par défaut à &FFFFFFFF si non présent)  

Formule de lecture, en considérant que MaskSizeCartridge représente la taille totale-1 de la rom accessible via la carte :  
`AdresseLecture = ((CpcBusAddress AND GGRO[CurrentGroup].GMSK) + GGRO[CurrentGroup].BaseAddress`) and MaskSizeCartridge

`CurrentGroup` résulte d'une commande **GROUPSELECT** passée à la carte.

Voir le fichier d'exemples dans ce dossier.

---

# Conclusion

Cette organisation hiérarchique des ROMs et groupes dans les fichiers CRO permet :  
- de simuler efficacement la cartouche complète dans la RAM,  
- de gérer les ROMs non contiguës,  
- de maintenir la compatibilité avec les anciennes et nouvelles générations de CPC,  
- d'optimiser les performances et l'accès aux données dans des cartouches dépassant 512k.  

L'utilisation des masques, des groupes et de la RAM permet de créer des systèmes modulables et rapides pour la lecture des ROMs dans le CPC PLUS et les cartes modernes comme C4CPC ou PICO GX.




