# FB_GripperState
```iecst
FUNCTION_BLOCK FB_GripperState
VAR_INPUT
	(* Default Input *)
	Enable				: BOOL;
	(* User Defined Inputs *)
	thOpen				: WORD := 50;
	thClose 			: WORD := 950;
	thPartMin 			: WORD := 800;
	thPartMax 			: WORD := 860;
END_VAR
VAR_IN_OUT
	hw					: UA_Schunk_mms;
END_VAR
VAR_OUTPUT
	(* Default Outputs *)
	InOperation			: BOOL;
	Error				: BOOL;
	(* User Outputs *)
	IsOpen				: BOOL;
	IsClosed			: BOOL;
	PartPresent			: BOOL;
END_VAR
VAR
	eInOperationBase	: E_InOperationBase := E_InOperationBase.Idle;
	eInOpGripper		: E_InOpGripper := E_InOpGripper.IsIdle;
	tonIdleCondition	: TON;
END_VAR

```

Code
```iecst
(*
	Input management. While gripper is moving, it must not be seen as an Idle Condition
	Here we use a timer as Input Management.
*)

tonIdleCondition(IN := (((hw.Value > thOpen) AND (hw.Value < thPartMin))  OR
		                ((hw.Value < thClose) AND (hw.Value > thPartMax))),
				 PT := T#1S);

(*
	Main State Machine
*)
CASE eInOperationBase OF
	E_InOperationBase.Idle :
		IF Enable THEN
			eInOperationBase := E_InOperationBase.Init;	
		END_IF
		
	E_InOperationBase.Init :
		IF NOT Enable THEN
			eInOperationBase := E_InOperationBase.Idle;
		ELSIF tonIdleCondition.Q THEN
		    eInOpGripper := E_InOpGripper.IsIdle;
		 	eInOperationBase := E_InOperationBase.Error;
		ELSE // Init internal state machine and jump in it
			IF hw.Value < thOpen THEN
				eInOpGripper := E_InOpGripper.IsOpen;
			ELSIF hw.Value > thClose THEN
				eInOpGripper := E_InOpGripper.IsClosed;
			ELSE
				eInOpGripper := E_InOpGripper.PartPresent;
			END_IF
			eInOperationBase := E_InOperationBase.InOp;
		END_IF
		
	E_InOperationBase.InOp :
		(*
			Sub State Machine Here
		*)
		// Machine initilized in E_InOperationBase.Init
		// Error condition again, air pressure could be removed while machine is running
		IF NOT Enable THEN
			eInOperationBase := E_InOperationBase.Idle;
		ELSIF tonIdleCondition.Q THEN			// Error condition
		    eInOpGripper := E_InOpGripper.IsIdle;
		 	eInOperationBase := E_InOperationBase.Error;
		ELSE // Init internal state machine and jump in it
			CASE eInOpGripper OF
				E_InOpGripper.IsIdle :
					; // Do nothing, condition is checked in IF
				E_InOpGripper.IsOpen :
					IF hw.Value > thClose THEN
						eInOpGripper := E_InOpGripper.IsClosed;
					ELSIF (hw.Value > thPartMin) AND (hw.Value < thPartMax) THEN
						eInOpGripper := E_InOpGripper.PartPresent;
					END_IF
				E_InOpGripper.IsClosed :
					IF hw.Value < thOpen THEN
						eInOpGripper := E_InOpGripper.IsOpen;
					END_IF
				E_InOpGripper.PartPresent :
					IF hw.Value < thOpen THEN
						eInOpGripper := E_InOpGripper.IsOpen;
					END_IF
			END_CASE
		END_IF
	E_InOperationBase.Error :
		IF NOT Enable THEN
			eInOperationBase := E_InOperationBase.Idle;
		END_IF
END_CASE

(*
	Output Management
*)
InOperation			:= (eInOperationBase = E_InOperationBase.InOp);
Error				:= (eInOperationBase = E_InOperationBase.Error);

IsOpen				:= (eInOperationBase = E_InOperationBase.InOp) AND (eInOpGripper = E_InOpGripper.IsOpen);
IsClosed			:= (eInOperationBase = E_InOperationBase.InOp) AND (eInOpGripper = E_InOpGripper.IsClosed);
PartPresent			:= (eInOperationBase = E_InOperationBase.InOp) AND (eInOpGripper = E_InOpGripper.PartPresent);

```

# FB_CloseGripper

```iecst
FUNCTION_BLOCK FB_CloseGripper
VAR_INPUT
	Execute		    : BOOL;
	thClosedMin		: WORD := 950;
END_VAR
VAR_IN_OUT
	hwSensor 		: UA_Schunk_mms;
	hwEV 			: UA_Festo;
END_VAR
VAR_OUTPUT
	Done			: BOOL;
	Active			: BOOL;
	Error			: BOOL;
END_VAR
VAR
	tnCheckDone		: TON;
	rExecute		: R_TRIG;
	eExecuteDone	: E_ExecuteDone;
END_VAR

```
Code
```iecst
(* 
	Manage Inputs
*)
rExecute(CLK := Execute);
// Timer.Q true if Execute and not threshold for more thant 1 sec.
tnCheckDone(IN := Execute AND NOT (hwSensor.Value > thClosedMin),
	        PT := T#1S);

CASE eExecuteDone OF
	E_ExecuteDone.Idle :
		IF rExecute.Q THEN
			eExecuteDone := E_ExecuteDone.Init;
		END_IF

	E_ExecuteDone.Init :
		// No init
		IF tnCheckDone.Q THEN
			eExecuteDone := E_ExecuteDone.Error;
		ELSE
			eExecuteDone := E_ExecuteDone.InOp;
		END_IF

	E_ExecuteDone.InOp :
		IF tnCheckDone.Q THEN
			eExecuteDone := E_ExecuteDone.Error;
		ELSIF (hwSensor.Value > thClosedMin) THEN
			eExecuteDone := E_ExecuteDone.Done;
		END_IF
	
	E_ExecuteDone.Done :
		IF NOT Execute THEN
			eExecuteDone := E_ExecuteDone.Idle;
		END_IF

	E_ExecuteDone.Error :
		IF NOT Execute THEN
			eExecuteDone := E_ExecuteDone.Idle;
		END_IF
END_CASE

IF eExecuteDone = E_ExecuteDone.InOp THEN
	hwEV.SetOut := FALSE;
END_IF

Done := (eExecuteDone = E_ExecuteDone.Done);
Active := (eExecuteDone = E_ExecuteDone.Init) OR (eExecuteDone = E_ExecuteDone.InOp) ;
Error := (eExecuteDone = E_ExecuteDone.Error);
```

# FB_OpenGripper
```iecst
FUNCTION_BLOCK FB_OpenGripper
VAR_INPUT
	Execute			: BOOL;
	thOpenMax		: WORD := 50;
END_VAR
VAR_IN_OUT
	hwSensor 		: UA_Schunk_mms;
	hwEV 			: UA_Festo;
END_VAR
VAR_OUTPUT
	Done			: BOOL;
	Active			: BOOL;
	Error			: BOOL;
END_VAR
VAR
	tnCheckDone		: TON;
	rExecute		: R_TRIG;
	eExecuteDone	: E_ExecuteDone;
END_VAR

```

Code
```iecst
(* 
	Manage Inputs
*)
rExecute(CLK := Execute);
// Timer.Q true if Execute and not threshold for more thant 1 sec.
tnCheckDone(IN := Execute AND NOT (hwSensor.Value < thOpenMax),
	        PT := T#1S);

CASE eExecuteDone OF
	E_ExecuteDone.Idle :
		IF rExecute.Q THEN
			eExecuteDone := E_ExecuteDone.Init;
		END_IF

	E_ExecuteDone.Init :
		// No init
		IF tnCheckDone.Q THEN
			eExecuteDone := E_ExecuteDone.Error;
		ELSE
			eExecuteDone := E_ExecuteDone.InOp;
		END_IF

	E_ExecuteDone.InOp :
		IF tnCheckDone.Q THEN
			eExecuteDone := E_ExecuteDone.Error;
		ELSIF (hwSensor.Value < thOpenMax) THEN
			eExecuteDone := E_ExecuteDone.Done;
		END_IF
	
	E_ExecuteDone.Done :
		IF NOT Execute THEN
			eExecuteDone := E_ExecuteDone.Idle;
		END_IF

	E_ExecuteDone.Error :
		IF NOT Execute THEN
			eExecuteDone := E_ExecuteDone.Idle;
		END_IF
END_CASE

IF eExecuteDone = E_ExecuteDone.InOp THEN
	hwEV.SetOut := TRUE;
END_IF

Done := (eExecuteDone = E_ExecuteDone.Done);
Active := (eExecuteDone = E_ExecuteDone.Init) OR (eExecuteDone = E_ExecuteDone.InOp) ;
Error := (eExecuteDone = E_ExecuteDone.Error);
```

# In Program

```iecst
PROGRAM PRG_Gripper
VAR
	(**********************************************************************************************************************
    	YOUR VARIABLES HERE
	**********************************************************************************************************************)
	fbGripperState		   : FB_GripperState;
	fbOpenGripper		   : FB_OpenGripper;
    fbCloseGripper	       : FB_CloseGripper;	
```

code
```iecst
(**********************************************************************************************************************
    YOUR CODE HERE
**********************************************************************************************************************)
fbGripperState.Enable := (stPlcOpenFbs.bEnableRemote)        		OR
                   		 (NOT stPlcOpenFbs.bEnableRemote 		AND 
				   		  NOT (fbPackStates.state.Aborting OR
                               fbPackStates.state.Aborted)); 
fbGripperState(hw := GVL_Abox.uaAboxInterface.uaSchunk);

stTestFbGripperHmi.gripperStateClosed := fbGripperState.IsClosed;
stTestFbGripperHmi.gripperStateOpen := fbGripperState.IsOpen;
stTestFbGripperHmi.gripperStatePartPresent := fbGripperState.PartPresent;
stTestFbGripperHmi.gripperStateError := fbGripperState.Error;
stTestFbGripperHmi.gripperStateInOp := fbGripperState.InOperation;

fbOpenGripper.Execute := (stStateMachineInfo.eState = E_Execute.eMotionBackDone) OR (eStarting = E_Starting.eMotionStartingDone);
fbCloseGripper.Execute := (stStateMachineInfo.eState = E_Execute.eMotionFwdDone);

fbOpenGripper(hwEV := GVL_Abox.uaAboxInterface.uaSchunkGripper,
	          hwSensor := GVL_Abox.uaAboxInterface.uaSchunk,
			  Done => stTestFbGripperHmi.executeOpenDone);
			  
fbCloseGripper(hwEV := GVL_Abox.uaAboxInterface.uaSchunkGripper,
	          hwSensor := GVL_Abox.uaAboxInterface.uaSchunk,
			  Done => stTestFbGripperHmi.executeCloseDone);
			  
GripperIsOpen   := fbOpenGripper.Done;
GripperIsClosed	:= fbCloseGripper.Done;
```