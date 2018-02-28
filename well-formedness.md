IELE Contract Well-Formedness
=============================

The following document describes a semantics of type- and semantic-checking in IELE. The semantics takes a contract as input and succeeds only if the contract is determined to be well-formed, i.e., free from type errors and malformed instructions or functions.

```k
require "iele-syntax.k"
require "data.k"
```

```k
module IELE-WELL-FORMEDNESS
    imports IELE-COMMON
    imports IELE-DATA
    imports DOMAINS
```

Configuration
-------------

The semantic checker for IELE has its own configuration separate from the configuration of execution. This is consistent with the semantics of other languages defined in K, which can have separate compile-time and execution-time semantics.

```k
    syntax IeleName ::= "Main" [token]

    configuration <k> $PGM:Contract </k>
                  <exit-code exit=""> 1 </exit-code>
                  <contracts> .Set </contracts>
                  <currentContract>
                    <types> intrinsicTypes </types>
                    <contractName> Main </contractName>
                    <declaredContracts> .Set </declaredContracts>
                    <functionBodies> .K </functionBodies>
                    <currentFunction>
                      <functionName> deposit </functionName>
                      <labels> .Set </labels>
                      <instructions> .K </instructions>
                    </currentFunction>
                  </currentContract>
```

Types
-----

IELE is a primarily untyped language, and therefore identifiers have one of two types: either an integer or a function. 

```k
    syntax Type ::= "int" | Types "->" ReturnType [klabel(funType)]
    syntax Types ::= List{Type,","} [klabel(typeList)]
    syntax ReturnType ::= Types | "unknown"
    syntax priorities typeList > funType

    syntax Types ::= ints(Int) [function]
 // -------------------------------------
    rule ints(0) => .Types
    rule ints(N) => int , ints(N -Int 1) [owise]
```

Contracts
---------

```k
    rule CONTRACT1::ContractDefinition CONTRACT2::ContractDefinition CONTRACTS => CONTRACT1 ~> CONTRACT2 CONTRACTS
    rule CONTRACT::ContractDefinition .Contract => CONTRACT

    rule <k> contract NAME { DEFINITIONS } => checkName(NAME) ~> DEFINITIONS ... </k>
         <contracts> CONTRACTS => CONTRACTS SetItem(NAME) </contracts>
         (_:CurrentContractCell => <currentContract>
           <contractName> NAME </contractName>
           ...
         </currentContract>)
      requires notBool NAME in CONTRACTS

    rule DEF::TopLevelDefinition DEFS => DEF ~> DEFS
    rule <k> .TopLevelDefinitions => BODIES ... </k>
         <functionBodies> BODIES </functionBodies>
         <types> ... init |-> _ -> .Types </types>

    rule <k> .K </k> <exit-code> 1 => 0 </exit-code>
```

Top Level Definitions
---------------------

```k
    rule <k> external contract NAME => . ... </k>
         <contracts> CONTRACTS </contracts>
         <declaredContracts> DECLARED => DECLARED SetItem(NAME) </declaredContracts>
      requires NAME in CONTRACTS andBool notBool NAME in DECLARED

    rule <k> (@ NAME = _)::GlobalDefinition => checkName(NAME) ... </k>
         <types> TYPES => TYPES NAME |-> int </types>
      requires notBool NAME in_keys(TYPES)

    rule <k> define @ init ( ARGS::LocalNames ) { BLOCKS } => . ... </k>
         <types> TYPES => TYPES init |-> ints(#sizeNames(ARGS)) -> .Types </types>
         <functionBodies> .K => function(init) ~> BLOCKS ... </functionBodies>
      requires notBool init in_keys(TYPES)

    rule <k> define @ NAME ( ARGS::LocalNames ) { BLOCKS } => checkName(NAME) ~> checkArgs(ARGS) ... </k>
         <types> TYPES => TYPES NAME |-> (ints(#sizeNames(ARGS)) -> unknown) </types>
         <functionBodies> .K => function(NAME) ~> BLOCKS ... </functionBodies>
      requires notBool NAME in_keys(TYPES) andBool NAME =/=K init

    rule <k> define public @ NAME ( ARGS::LocalNames ) { BLOCKS } => checkName(NAME) ~> checkArgs(ARGS) ... </k>
         <types> TYPES => TYPES NAME |-> ints(#sizeNames(ARGS)) -> unknown </types>
         <functionBodies> .K => function(NAME) ~> BLOCKS ... </functionBodies>
      requires notBool NAME in_keys(TYPES) andBool NAME =/=K init

    syntax KItem ::= function(IeleName)
 // -----------------------------------
    rule <k> function(NAME) => . ... </k>
         (_:CurrentFunctionCell => <currentFunction>
           <functionName> NAME </functionName>
           ...
         </currentFunction>)

    syntax Int ::= #sizeNames(LocalNames) [function]
 // ------------------------------------------------
    rule #sizeNames(.LocalNames) => 0
    rule #sizeNames(N , NAMES) => 1 +Int #sizeNames(NAMES)

    syntax KItem ::= checkArgs(LocalNames)
                   | checkNameArgs(LocalNames)
                   | checkIntArgs(LocalNames, Int)
 // ----------------------------------------------
    rule checkArgs(.LocalNames) => .
    rule checkArgs(% N:NumericIeleName , ARGS) => checkIntArgs(% N, ARGS, 0)
    rule checkArgs(% N, ARGS) => checkNameArgs(ARGS) requires notBool isNumericIeleName(N)

    rule checkNameArgs(% N, ARGS) => checkName(N) ~> checkNameArgs(ARGS) requires notBool isNumericIeleName(N)
    rule checkNameArgs(.LocalNames) => .

    rule checkIntArgs( % N , ARGS , I) => checkIntArgs(ARGS, I +Int 1)
      requires String2Int(IeleName2String(N)) ==Int I
    rule checkIntArgs(.LocalNames, _) => .
```

Blocks
------

```k
    rule <k> BLOCK:UnlabeledBlock BLOCKS => BLOCKS ... </k>
         <instructions> .K => BLOCK ... </instructions>
    rule BLOCK::LabeledBlock BLOCKS => BLOCK ~> BLOCKS
    rule <k> .LabeledBlocks => INSTRS ... </k>
         <instructions> INSTRS </instructions>

    rule <k> NAME : BLOCK::Instructions => . ... </k>
         <labels> LABELS => LABELS SetItem(NAME) </labels>
         <instructions> .K => BLOCK ... </instructions>
      requires notBool NAME in LABELS

    rule INSTR::Instruction INSTRS::Instructions => INSTR ~> INSTRS
    rule .Instructions => .
```

Instructions
------------

### Regular Instructions

Each of these instructions takes some number of immediates, globals, or registers, and returns zero or one registers. Checking them is as straightforward as checking for reserved names.

```k
    syntax KResult ::= Operand
                     | Operands
    syntax NonEmptyOperands ::= Operands
 // ------------------------------------

    rule LVAL = OP1 => checkLVal(LVAL) ~> checkOperand(OP1)
    rule LVAL = load OP1 => checkLVal(LVAL) ~> checkOperand(OP1)
    rule LVAL = load OP1, OP2, OP3 => checkLVal(LVAL) ~> checkOperands(OP1, OP2, OP3)
    rule store OP1, OP2 => checkOperands(OP1, OP2)
    rule store OP1, OP2, OP3, OP4 => checkOperands(OP1, OP2, OP3, OP4)

    rule LVAL = sload OP1 => checkLVal(LVAL) ~> checkOperand(OP1)
    rule sstore OP1, OP2 => checkOperands(OP1, OP2)

    rule LVAL = iszero OP1 => checkLVal(LVAL) ~> checkOperand(OP1)
    rule LVAL = not    OP1 => checkLVal(LVAL) ~> checkOperand(OP1)

    rule LVAL = add OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = mul OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = sub OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = div OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = exp OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = mod OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)

    rule LVAL = addmod OP1, OP2, OP3 => checkLVal(LVAL) ~> checkOperands(OP1, OP2, OP3)
    rule LVAL = mulmod OP1, OP2, OP3 => checkLVal(LVAL) ~> checkOperands(OP1, OP2, OP3)
    rule LVAL = expmod OP1, OP2, OP3 => checkLVal(LVAL) ~> checkOperands(OP1, OP2, OP3)

    rule LVAL = byte OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = sext OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = twos OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)

    rule LVAL = and   OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = or    OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = xor   OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)
    rule LVAL = shift OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)

    rule LVAL = cmp _ OP1, OP2 => checkLVal(LVAL) ~> checkOperands(OP1, OP2)

    rule LVAL = sha3 OP1 => checkLVal(LVAL) ~> checkOperand(OP1)
    rule log OP1                     => checkOperand(OP1)
    rule log OP1, OP2                => checkOperands(OP1, OP2)
    rule log OP1, OP2, OP3           => checkOperands(OP1, OP2, OP3)
    rule log OP1, OP2, OP3, OP4      => checkOperands(OP1, OP2, OP3, OP4)
    rule log OP1, OP2, OP3, OP4, OP5 => checkOperands(OP1, OP2, OP3, OP4, OP5)

    rule revert OP1 => checkOperand(OP1)
    rule selfdestruct OP1 => checkOperand(OP1)
```

### Static Jumps

Checking these instructions requires checking that a label exists that matches the specified label.

```k
    rule <k> br NAME => . ... </k>
         <labels> ... SetItem(NAME) </labels>

    rule <k> br OP1, NAME => checkOperand(OP1) ... </k>
         <labels> ... SetItem(NAME) </labels>
```

### Function Calls and Returns

Checking these instructions requires checking the types of local function calls and checking the consistency of the return type.

```k
    rule <k> RETS = call @ NAME ( ARGS ) => checkLVals(RETS) ~> checkOperands(ARGS) ... </k>
         <types> ... NAME |-> ARGTYPES -> RETTYPES:Types </types>
      requires ints(#sizeRegs(ARGS)) ==K ARGTYPES andBool ints(#sizeLVals(RETS)) ==K RETTYPES

    rule <k> RETS = call @ NAME ( ARGS ) => checkLVals(RETS) ~> checkOperands(ARGS) ... </k>
         <types> ... NAME |-> ARGTYPES -> (unknown => ints(#sizeLVals(RETS))) </types>
      requires ints(#sizeRegs(ARGS)) ==K ARGTYPES

    rule STATUS, RETS = call @ NAME at OP1 ( ARGS ) send OP2 , gaslimit OP3 => checkLVals(STATUS, RETS) ~> checkOperands(OP1 , OP2 , OP3 , ARGS)
    rule STATUS, RETS = staticcall @ NAME at OP1 ( ARGS ) gaslimit OP2 => checkLVals(STATUS, RETS) ~> checkOperands(OP1 , OP2 , ARGS)

    rule <k> ret OPS => checkOperands(OPS) ... </k>
         <functionName> NAME </functionName>
         <types> ... NAME |-> _ -> RETTYPES:Types </types>
      requires ints(#sizeRegs(OPS)) ==K RETTYPES

    rule <k> ret OPS => checkOperands(OPS) ... </k>
         <functionName> NAME </functionName>
         <types> ... NAME |-> _ -> (unknown => ints(#sizeRegs(OPS))) </types>
```

### Contract Creation

Checking these instructions also requires checking that the contract they reference has been declared.

```k
    rule <k> STATUS , RET = create NAME ( ARGS ) send OP1 => checkLVals(STATUS, RET) ~> checkOperands(OP1 , ARGS) ... </k>
         <declaredContracts> ... SetItem(NAME) </declaredContracts>

    rule STATUS , RET = copycreate OP1 ( ARGS ) send OP2 => checkLVals(STATUS, RET) ~> checkOperands(OP1 , OP2 , ARGS)
```

```

Types of intrinsic functions
----------------------------

Below are defined the types of the reserved functions that begin with "iele.".

```k
    syntax Map ::= "intrinsicTypes" [function]
 // ------------------------------------------
    rule intrinsicTypes =>
    (iele.invalid |-> .Types -> .Types
    (iele.gas |-> .Types -> int, .Types
    (iele.gasprice |-> .Types -> int, .Types
    (iele.gaslimit |-> .Types -> int, .Types
    (iele.beneficiary |-> .Types -> int, .Types
    (iele.timestamp |-> .Types -> int, .Types
    (iele.number |-> .Types -> int, .Types
    (iele.difficulty |-> .Types -> int, .Types
    (iele.address |-> .Types -> int, .Types
    (iele.origin |-> .Types -> int, .Types
    (iele.caller |-> .Types -> int, .Types
    (iele.callvalue |-> .Types -> int, .Types
    (iele.msize |-> .Types -> int, .Types
    (iele.codesize |-> .Types -> int, .Types
    (iele.blockhash |-> int -> int, .Types
    (iele.balance |-> int -> int, .Types
    (iele.extcodesize |-> int -> int, .Types
    )))))))))))))))))
    
```

Reserved Names
--------------

All identifiers beginning with "iele." are reserved by the language and cannot be written to.

```k
    syntax K ::= checkName(IeleName) [function]
 // -------------------------------------------
    rule checkName(NAME) => .
      requires lengthString(IeleName2String(NAME)) <Int 5 orBool substrString(IeleName2String(NAME), 0, 5) =/=String "iele."

```

Checking Operands
-----------------

```k
    syntax KItem ::= checkOperand(Operand)
                   | checkOperands(Operands)
 // ----------------------------------------
    rule checkOperands(OP , OPS) => checkOperand(OP) ~> checkOperands(OPS)
    rule checkOperands(.Operands) => .

    rule checkOperand(% NAME) => .
    rule checkOperand(_:IntConstant) => .
    rule <k> checkOperand(@ NAME) => . ... </k>
         <types> ... NAME |-> int </types>
```

Checking LValues
----------------

```k
    syntax KItem ::= checkLVal(LValue)
                   | checkLVals(LValues)
 // --------------------------------------
    rule checkLVals(LVAL , LVALS) => checkLVal(LVAL) ~> checkLVals(LVALS)
    rule checkLVals(.LValues) => .

    rule checkLVal(% NAME) => checkName(NAME)
endmodule
```