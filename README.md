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
    <img src="./img/MMS 22-IO-Link.png"
         alt="Lost image : MMS 22-IO-Link">
    <figcaption>Schunk MMS 22-IO-Link</figcaption>
</figure>

> Ce capteur utilise la technologie **IO-Link**. Il est utilisé pour déterminer la position d'ouverture d'un actioneur pneumatique.

> Ce qui nous intéresse au niveau de la programmation PLC, c'est le format de donnée qu'il fournit.

# Gripper

<figure>
    <img src="./img/MGM-plus 40.png"
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

# Control Module Gripper
Nous allons contsruire un CM Control Module Gripper constitué de deux Function Blocks.

Le premier pilote un actuateur, le deuxième pilote un contrôle.

## Actuateur
L'actuateur fonctionne selon le principe Execute. C'est à dire que la fonction est activée sur le flanc montant d'un signal Execute.

Le module utilisé est relativement basique, mais dans son principe, il correspond à ce que l'on retrouvera dans un travail pratique utlérieur pour piloter une commande d'axe électrique composée de plusieurs régulateurs en cascade.

## Sensor
Ici aussi, le sensor est relativement basique, mais comme on l'a vu dans le travail pratique précédent, on peut rapidement augementer la complexité de la partie logicielle afin de rendre l'utilisation de ce capteur polyvalente. Ici, nous allons utiliser ce capteur non seulement pour définir si l'actuateur est ouvert ou fermé, mais aussi pour déterminer si une pièce a été saisie correctement.

## hardware
```iecst

TYPE HW_GripperSchunk_typ :
STRUCT
	actuator	: UA_Festo;
	sensor		: UA_Schunk_mms;
END_STRUCT
END_TYPE

```



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

## Help

**FUNCTION_BLOCK TP**

Implements a pulse timer

```iecst
(* Example declaration *)
TPInst : TP ;

(* Example in ST *)
TPInst(IN := VarBOOL1, PT:= T#5s);
VarBOOL2 := TPInst.Q;
```

<figure>
    <img src="./puml/TpTimeDiagram/TpTimeDiagram.svg"
         alt="Lost image :TpTimeDiagram.svg">
    <figcaption>Pulse Timer diagram</figcaption>
</figure>

|Scope  |Name |Type |Comment|
|-------|-----|-----|-------|
|Input	|IN	  |BOOL	|Rising edge starts the pulse timer and sets Q to TRUE|
|Input	|PT	  |TIME	|Length of the pulse (high-signal)|
|Output	|Q	  |BOOL	|Pulse signal, set to TRUE for PT milliseconds if EN has a rising edge|
|Output	|ET	  |TIME	|Elapsed time since pulse timer started. It will then remain constant after PT is reached|

# Quelques liens
[IO-Link, site officiel](https://io-link.com)
[IODDfinder](https://ioddfinder.io-link.com)


# Solution (partial) for FUNCTION_BLOCK FB_CheckGripperState

```iecst
// Header
(*
	www.hevs.ch
	Institut Systemes Industriels
	Project: 	Projet No: PW_04
	Author:		Cedric Lenoir
	Date:		2024 March 4
	
	Summary:	Structure for the harware of CM Gripper
*)
FUNCTION_BLOCK FB_CheckGripperState
VAR_INPUT
	Enable				: BOOL;
	// Position of sensor when gripper is closed : should be 1000, for exemple: 990
	thClosed			: UINT;
	// Position of sensor when gripper is open : should be 0, for example: 10
	thOpen				: UINT;
	// Position of sensor when closed with part present, should be 787, for example 750
	thParlLowLimit		: UINT;
	// Position of sensor when closed with part present, should be 787, for example 800
	thPartHighLimit		: UINT;
END_VAR
VAR_IN_OUT
	hwSensor			: UA_Schunk_mms;
	hwActuator			: UA_Festo;
END_VAR
VAR_OUTPUT
	InOperation			: BOOL;
	Error				: BOOL;
	ErrorID				: WORD;
END_VAR
VAR
	eEnableInOperation	: E_EnableInOperation;
	// Do not control while gripper state change
	lockCheck			: TP;
	lastActuatorValue	: BOOL;
END_VAR
VAR CONSTANT
	// 
	GripperNoError		: WORD := 16#0001;
	GripperNotOpen		: WORD := 16#0002;
END_VAR

// Code

(*
	Principle:
	If gripper active without part, value of sensor should be closed with value > 990
	If gripper active with part, value should be 750 < value < 800
	If gripper not active, value should be < 10
	
	The control does not cover all cases.
*)

(*
	Input management, set gripper transition
	That is, we give pt time to the gripper to go to stable position
*)
lockCheck(IN := hwActuator.SetOut <> lastActuatorValue,
	      PT := T#500MS);
lastActuatorValue := hwActuator.SetOut;

CASE eEnableInOperation OF
	    // Starting state of a function block
	eEnableInOperation.STATE_IDLE :
		ErrorID := GripperNoError;
		IF Enable THEN
			eEnableInOperation := E_EnableInOperation.STATE_INIT;
		END_IF
    // Initialization of the function block State
    eEnableInOperation.STATE_INIT :
		ErrorID := GripperNoError;
		// Nothing to do in this state
		IF NOT Enable THEN
			eEnableInOperation := E_EnableInOperation.STATE_IDLE;
		ELSE
			eEnableInOperation := E_EnableInOperation.STATE_INOP;
		END_IF
    //  Working state of the function block
    eEnableInOperation.STATE_INOP :
		(*
			Check in case of actuator not active
			gripper should be open, sensor 0
		*)
		IF hwActuator.SetOut AND 
           NOT lockCheck     AND
		   NOT hwSensor.Value < thOpen THEN
		   ErrorID := GripperNotOpen;
		   eEnableInOperation := E_EnableInOperation.STATE_ERROR;
		END_IF
		
		// To be completed and checked
		
    // State is active after an error occurs
    eEnableInOperation.STATE_ERROR :
		;
END_CASE
```