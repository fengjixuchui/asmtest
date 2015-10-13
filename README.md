# asmtest
Example for Writing Proper AMD64 Assembly Code for Windows

Goal of this small project is to grab all information available on x64 assembly from Microsoft, Intel Developer Zone,
Codemachine and McDermott Cypersecurity on x64 assembly, and put it together in one assembly file such that everyone can easily look or understand how to properly interface with the Windows 64-bit ABI.

Core elements include how to properly handle stack-based data.

Syntax used:

    r64 = any 64-bit register (rax, rbx, rcx, rdx, r8, r9, r10, r11, r12, r13, r14, r15, rsp, rbp, rsi, rdi)
    m64 = any 64-bit value in memory (*pUlonglong)
    imm64 = any hardcoded 64-bit value, e.g. "0CCCCCCCCCCCCCCCCh"


Conclusion:

- Inside of each call, IMMEDIATELY after a call instruction has been executed, each CALLEE is guaranteed to receive a stack pointer "rsp" = 0x???????????????8. It does not matter if the call is API-internal or in a user's program.

- There exist 2 types of functions, "LEAF FUNCTIONS and "FRAME FUNCTIONS".

- A LEAF function does not call any other functions (--> it is a leaf on a tree of function calls) or issue any Intel 0F 05 syscalls. After a possible function PROLOGUE the stack pointer is NOT required to be 16-bit aligned.

- A leaf function is NOT required to have a function prologue.

- Using local variables within a leaf function requires you to write a "sub rsp, XYZ" function.
- XYZ is given by: THE SUM OF ALL VARIABLE SIZES, ROUNDED UP TO AN 8 BYTE ALIGNED VALUE.
- Before exiting the function you must write an "add rsp, XYZ" instruction.

- In a leaf function THE FIRST VARIABLE is found at "rsp + 0". Subsequent ones at "rsp + sizeOfFirstVariable".

- Existence of a function PROLOGUE requires always a corresponding EPILOGUE:
- --> Each previous "sub rsp, XYZ" must now be matched with a "add rsp, XYZ", where "XYZ" must be the VERY SAME value.
- --> Each previous "push r64" must now be matched with a "pop r64", where "r64" must be the VERY SAME register.
- --> "push r64" before "sub rsp, XYZ" and "add rsp, XYZ" before "pop r64".
Example:

someFunction PROC

    push rsi    ;Prologue
    push rdi
    sub rsp, 18h
    nop     ;Actual calculations
    add rsp, 18h    ;Epilogue
    pop rdi
    pop rsi
    ret

someFunction ENDP

- The following registers must be considered destroyed after a function call:
- --> rcx, rdx, r8, r9, rax
- The following registers must be preserved by callee:
- --> rbx, rbp, rsi, rdi, r12, r13, r14, r15
- The following registers are used/altered by a syscall instruction:
- --> r10, rdx, r8, r9 (all used and altered), r10, r11 (altered)

- both leaf and frame functions can HOME up to 4 ARBITRARY 8-BYTE (r64, imm64, m64) values. (SHADOW SPACE):

someFunction PROC

    mov [rsp+8], rcx
    mov rcx, 0AFCEFCEFCEFCEFCEh
    mov [rsp+10h], rcx
    mov rcx, [rsp-0A68h]
    mov [rsp+18h], rcx
    mov dword ptr [rsp+20h], 065AFEBCAh
    nop
    ...

someFunction ENDP

Often this is used to save rcx, rdx, r8 and r9, it is most common in debug version of C code:


someFunction2 PROC

    mov [rsp+8], rcx
    mov [rsp+10h], rdx
    mov [rsp+18h], r8
    mov [rsp+20h], r9
    nop
    ...

someFunction2 ENDP


- A FRAME function calls other functions or issues syscall instructions, and MUST employ a function prologue. After execution of the prologue, the stack pointer MUST be 16-bit aligned, say, it must look like 0x???????????????0!

- Any frame function must perform a MINIMUM allocation of "sub rsp, 28h" if there is NO or an even count "push r64" before. 
- --> The allocation is for SHADOW SPACE (see above) and return address of the next deeper callee.

someSmallFrameFunction PROC

    sub rsp, 28h
    call nextDeeperFunction
    add rsp 28h
    ret
    
someSmallFrameFunction ENDP


Even count:

someSmallFrameFunction2 PROC

    push rbx
    push rsi
    sub rsp, 28h
    call nextDeeperFunction
    add rsp 28h
    pop rsi
    pop rbx
    ret
    
someSmallFrameFunction2 ENDP


- Any frame function must perform a MINIMUM allocation of "sub rsp, 30h" if there is an ODD count of "push r64" before.
- --> The allocation is for shadow space, return address and the required 8-byte alignment of the stack pointer

someSmallFrameFunction3 PROC

    push rbx
    sub rsp, 28h
    call nextDeeperFunction
    add rsp 28h
    ret
    
someSmallFrameFunction3 ENDP

- Space for additional arguments to the callee  

- A function PROLOGUE of a leaf function can be completely missing.
- at minimum look like:

someSmallSub PROC

    xor eax, eax
    ret
    
someSmallSub ENDP

and at maximum like (following function allocates space on the stack for 3 8 byte local variables):

someBigSub PROC

    mov [rsp+8], rcx
    mov [rsp+10h], rdx
    mov [rsp+18h], r8
    mov [rsp+20h], r9
    push rbx
    push rbp
    push rsi
    push rdi
    push r12
    push r13
    push r14
    push r15
    sub rsp, 18h
    mov qword ptr [rsp], 1
    mov qword ptr [rsp+8], 2
    mov qword ptr [rsp+10h], 3
    mov eax, 12345678h
    add rsp, 18h
    pop r15
    pop r14
    pop r13
    pop r12
    pop rdi
    pop rsi
    pop rbp
    pop rbx
    ret
  
someBigSub ENDP



  looks like
- After a function PROLOG 
