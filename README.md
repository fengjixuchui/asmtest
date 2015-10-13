# asmtest
Example for Writing Proper AMD64 Assembly Code for Windows

Goal of this small project is to grab all information available on x64 assembly from Microsoft, Intel Developer Zone,
Codemachine and McDermott Cypersecurity on x64 assembly, and put it together in one assembly file such that everyone can easily look or understand how to properly interface with the Windows 64-bit ABI.

Core elements include how to properly handle stack-based data.


Conclusion:

- Inside of each call, IMMEDIATELY after a call instruction has been executed, each CALLEE is guaranteed to receive a stack pointer "rsp" = 0x???????????????8. It does not matter if the call is API-internal or in a user's program.
- There exist 2 types of functions, "LEAF FUNCTIONS and "FRAME FUNCTIONS".
- A function PROLOG of a leaf function can be at minimum look like:
someSmallSub PROC
  xor eax, eax
  ret
someSmallSub ENDP

and at maximum like:
someBigSub PROC
  
  xor eax, eax
  ret
someBigSub ENDP


  looks like
- After a function PROLOG 
