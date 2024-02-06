<h1 align="left">
  <br>
  <img src="./img/hei-en.png" alt="HEI-Vs Logo" width="350">
  <br>
  HEI-Vs Engineering School - Industrial Automation Base
  <br>
</h1>

Cours AutB

Author: [Cédric Lenoir](mailto:cedric.lenoir@hevs.ch)

# LAB 04 A function block for an actuator

Dans ce travail, nous allons construire un gripper, ou pince.

Avec un système de programmation empirique on pourrait se dire que tout ce dont nous avons besoin pour un capteur, c'est une entrée analogique ou digitale, donc en fin de compte, un ``REAL`` ou un ``BOOL``.

Dans la pratique, une simple entrée ou sortie sera entourée d'une logique qui permettra de la mettre en forme et de la valider. Afin d'éviter de réécrire la même logique pour chaque entrée et chaque sortie, nous allons encapsuler l'ensemble dans un bloc. Le Bloc fonctionnel.

# Commentaires de rédaction.
Il faudra voir à la fin de la rédaction si il faut "Splitter ce TP en Sensor et Actuator".

Encore à réfléchir, mais deux FB complets

Au pire, si robust programming n'est pas vu, on indique comment on fait et la théorie suivra la pratique.

Ce serait bien de proposer un canevas pour Enable et Execute, qui seront vus ou pas dans Robust Programming.

# Sensor
Dans le cadre de ce travail pratique, nous utilisons un capteur à effet hall d'origine Schunk.

<figure>
    <img src="./img/MMS 22-IO-Link.webp"
         alt="Lost image : MMS 22-IO-Link">
    <figcaption>Schunk MMS 22-IO-Link</figcaption>
</figure>

> Ce capteur utilise la technologie **IO-Link**. Il est utilisé pour déterminer la position d'ouverture d'un actioneur pneumatique.

# Gripper

<figure>
    <img src="./img/MGM-plus 40.webp"
         alt="Lost image : MGM-plus 40.webp">
    <figcaption>Schunk MGM-plus 40</figcaption>
</figure>

> Noter les deux vis en dessus du logo Schunk qui servent à maintenir le capteur.

## Données techniques
Pince pour petites pièces MPG-plus, Taille: 40, pneumatique

|Intitulé                           |Valeurs|
|-----------------------------------|-------------|
|Course par doigt| 6 mm|
|Force à la fermeture| 135 N|
|Force à l'ouverture| 110 N|
|Température ambiant maxi.          | 90 °C|

> Ces données techniques concernent principalement la personne qui va gérer le hardware.

> Ce qui nous intéresse au niveau de la programmation PLC, c'est le format de donnée qu'il fournit.

## Données synchrones.
Les données **synchrones** sont celles qui sont transmises à interval de temps plus ou moins fixes. On les retrouve dans le tableau ci-dessous.



## Données asynchrones.
Les données **asynchrones**, sont celles qui permettent soit de lire des caractéristiques du capteur. Soit de le paramétrer. Ci-dessous, une partie des paramètres sont montrés à titre d'exemple.

Les [paramètres pré définis](#io-link-pre-defined-parameters) permettent d'identifier le capteur. Les [paramètres des canaux binaires](#io-link-binary-data-channels) permettent de gérer un seuil, par exemple une détection, directement dans le capteur.


### IO-Link Pre defined parameters pour 
|Index      |Subindex (dec) |Access |Parameter name |Coding |Definition|
|-----------|---------------|-------|---------------|-------|----------|
|0x000C (12)| 0             |R/W    |Device Access Locks| Uint16|0: Unlocked (default)
|           |               |       |             |       |1: Device is operating properly|
|0x0010 (16)| 0             |R      |Vendor Name| String|Baumer Electric AG|
|0x0011 (17)| 0             |R      |Vendor Text| String|www.baumer.com|
|0x0012 (18)| 0             |R      |Device Name| String|Product Key External (<Product Key Internal>)|
|0x0013 (19)| 0             |R      |Product Id| String|Baumer Article Number|
|0x0014 (20)| 0             |R      |Device Text| String|Sensor specific|
|0x0015 (21)| 0             |R      |Serial Number| String|Production Order Nr / Serial Nr |
|0x0017 (23)| 0             |R      |Firmware Revision| String|Major.Minor “##.##”|
|0x0018 (24)| 0             |R/W    |Application Specific Tag|String| Default: Filled with ******, as recommended by the IO-Link spec.|
|0x0024 (36)| 0             |R      |Device Status| Uint16| 0: Device is operating properly
|           |               |       |             |       | 1: Device is operating properly
|           |               |       |             |       | 2: Out-of-Specification
|           |               |       |             |       | 3: Functional-Check
|           |               |       |             |       | 4: Failure
|           |               |       |             |       | 5 - 255: Reserved|
|0x0025 (37)| 0             |R      |Detailed Device Status| Uint16| EventQualifier “0x00” EventCode “0x00, 0x00”

### IO-Link Binary Data Channels
|Index      |Subindex (dec) |Access |Parameter name| Coding| Definition|
|-----------|---------------|-------|---------------|-------|----------|
|0x003c (60)| 01            |R/W    | Setpoint SP1  |Uint16 |Teach Point [mm] (TP)|
|           | 02            |R/W    |Setpoint SP2   |Uint16 |Not supported|
|0x003d (61)| 01            |R/W    |Switchpoint logic|Uint8|0x00: not inverted 0x01: inverted|
|           | 02            |R/(W)  |Switchpoint mode|Uint8|Fixed value 0x01: Single point mode|

La gestion des paramètres asynchrones peut se faire directement dans le PLC. Cependant, la nature du PLC, dédié à des tâches cycliques, le rend peu efficace pour ce genre de travail. Il existe, par exemple chez Baumer, un logiciel, [Baumer Sensor Suite](https://www.baumer.com/us/en/products/baumer-sensor-suite/a/baumer-sensor-suite) qui permet de paramétrer directement les capteus sans passer par le PLC.

Le logiciel Baumer BSS mentionné est toutefois lié à un type de IO-Link Master, l'appareil qui sert de passerelle entre le monde IO-Link et le PLC.

<figure>
    <img src="./img/IO-Link_Master_Profinet_8_Port_IP67.webp"
         alt="Lost image : IO-Link_Master_Profinet_8_Port_IP67">
    <figcaption>IO-Link_Master_Profinet_8_Port_IP67</figcaption>
</figure>

Le contenu de ce module est une partie de la réponse au problème du paramétrage du capteur depuis le PLC. Même si nous n'allons pas le faire, le développement d'un Fonction Block, même complexe, pourrait permettre ensuite de paramétrer tous les capteurs d'une installation et être réutilisé pour les futures installation.

# Travail pratique
Nous allons coder un Function Block qui lit les données synchrone du capteur. Les informations seront ensuite mises en forme et codées pour donner utiliser directement les informations utiles.

- Distance
- Erreur de mesure
- Seuil

## Description du Function Block

# Execute Done Base 

### Input

|Name   |Type       |Description|
|-------|-----------|-----------|
|Device |UA_O300_DL |In the particular context of the ctrlX to S7 interface.|
|Enable	|BOOL	      |Activate Function Block, set data value in output if valid.|

### Output
|Name         |Type         |Description         |
|-------------|:------------|--------------------|
|InOperation	|BOOL	        |Valid data at output|
|Error	      |BOOL	        |There is an error   |
|ErrorID	    |WORD         |Some details about the error with Error Code.|

|ErrorID Code |Description|
|-------------|-----------|
|16#0000      |Data valid |
|16#0001      |Quality bit, signal is below the configured threshold.|
|16#0002      |Alarm bit, signal an error in sensor, this alarm has priority over ID 16#0001|

## Comportement du Function Block
Le diagramme d'état d'un bloc fonctionnel de type Enable In Operation ressemble au digramme suivant: 

<figure>
    <img src="./puml/Lab03_EnableInOpBase/Lab03_EnableInOpBase.svg"
         alt="Lost image : Lab03_EnableInOpBase">
    <figcaption>Function Block Enable In Operation Base</figcaption>
</figure>

Dans le cas qui nous intéresse, nous voulons obtenir deux informations supplémentaires qui sont directement dépendantes de la machine d'état.
### Output
|Name         |Type         |Description         |
|-------------|:------------|--------------------|
|HighLimit	  |BOOL	        |Valid signal above HighThreshold|
|LowLimit	    |BOOL	        |Valid signal above LowThreshold |

On pourrait la représenter ainsi:
<figure>
    <img src="./puml/Lab03_EnableInOpBaseSubStates/Lab03_EnableInOpBaseSubStates.svg"
         alt="Lost image : Lab03_EnableInOpBaseSubStates">
    <figcaption>Function Block Enable In Operation Base with Sub States</figcaption>
</figure>

Il serait théoriquement possible de passer directement de LevelLow à LevelHigh, mais cela n'apporte rien au fonctionnement général et implique des transitions supplémentaire.
Par contre, cette forme pourrait s'avérer un peu plus complexe à coder, personnellement je ne l'utilise pour ainsi dire jamais. Je préfère coder la forme complète.

<figure>
    <img src="./puml/Lab03_EnableInOpBaseMoreStates/Lab03_EnableInOpBaseMoreStates.svg"
         alt="Lost image : Lab03_EnableInOpBaseMoreStates.svg">
    <figcaption>Function Block Enable In Operation Base with More States</figcaption>
</figure>

On constate très vite, que même si le nombre d'états est limité, le représentation complète devient vite difficle à déchiffrer.

### Conclusion
Utiliser le diagramme d'état composé, quitte à coder des états simples.

# Actuator
Principe
On veut utiliser le gripper pour saisir une pièce.
Le gripper est activé manuellement depuis un HMI Prosys Monitor.
On va utiliser le capteur du gripper pour détecter si la pièce est saisie correctement.
Si la pièce est saisie correctement, on active un flag à destination du Robot.
Si la pièce n'est pas considérée comme saisie correctement, on active un gripper.

Il faut considérer que si le FB_Gripper est inséré dans un programme, la commande Pick, Execute, pourrait poser un problème pour débloquer le système car le flag est dépendant du programme.


Il est donc nécessaire de prévoir un mode manuel ou recovery pour débloquer le système.


# Titiller les étudiants.
On peut ajouter ici, le cas de figure du Recovery, c'est à dire que si la pièce n'est pas prise correctement, comment va-t-on faire pour donner la possibilité de retirer manuellement la pièce défectueuse sans devoir couper l'alimentation en air de la machine.

# Titiller suite
Déterminer dans quel sens on va piloter le gripper selon différents cas de figure, normalement ouvert, normalement fermé.
On en arrive à un gripper paramétrable qui permette de choisir le type d'activation.

Un actuateur simple, le gripper.
La capteur Schunk sera aussi utilisé.

# CFG, Config
On peut ajouter une structure config en ``VAR_IN_OUT`` qui permettra non seulement de paramétrer le gripper, mais aussi les limites du sensor via le HMI.

# Les états
On va imposer les états minimaux pour l'écriture de la machine d'état.

## Execute State



## Modèle Enable InOperation

[Détail de EnableInOperation](E_EnableInOperation.md)

## Modèle Execute Done

[Détail de ExecuteDoneBase](E_ExecuteDoneBase.md)

# Quelques liens
[IO-Link, site officiel](https://io-link.com)
[IODDfinder](https://ioddfinder.io-link.com)
