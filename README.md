# C-style Language Compiler Targeting LC-3

## Lexer

The lexer is configured with a priority-ordered list of compiled regexes, each designed to match a corresponding token string. We match from the beginning of a given substring with all the token regexes, and when we find a match, we read off a token and add it to the accumulating token array.

The lexer sometimes will need to disambiguate when multiple token regexes find a match. It does this by finding the joint longest matches, and picking the unique joint longest match with the highest priority.

We also remove the matched token substring from the beginning of the byte stream, so that the process is ready for the next token to be read.

Currently, the lexer is maximising readability and extensibility. However, to make it more efficient there are some tactics that will likely help lower its runtime cost.

- Stop using regexes for single-character tokens and just check for them directly in some switch-case statement
- If none of those trigger, then check if the current character is alphabetic
  - If the current character is alphabetic then we can fall back on the token regex list approach, but the only token types in the list will be alphabetic tokens e.g. name tokens and keyword tokens
  - Or just greedily consume characters that might be part of a name or keyword. Once we reach a character that can't be part of a name or keyword, then we look back at the accumulated characters and try to discern which token it was.
- Then check for numeric characters. Currently no name token or keyword token is allowed to begin with a numeric character
  - In this case we could greedily consume numeric characters into an accumulating token, stopping when the current character is not numeric

## Parser

The parser is a hand-written recursive descent parser. The expression sector of the parser uses a variant of recursive descent with adjustments to handle infix operator precedence.

## Intermediate Representation

The intermediate representation is largely an abstracted/generic assembly language with extra utilities like function calls and struct handling operations.

The IR allows you to declare stack variable allocations with `ALLOCATE_STACK_SPACE`. For current purposes these are guaranteed to be translated into stack variables, which seems to be a detail that LLVM exposes in some cases (`alloca`). There are also "virtual registers", which are declared implicitly and are used in operations that take Three-Address-Code form. These "registers" are numbers prefixed with `#` because the lexer and parser would not allow that character through as part of a variable name ensuring disambiguation with any possible user variable.

The local variables declared on the stack don't need to be explicitly deallocated. This is handled by the backend, though in some cases we need to mark out scopes using `BLOCK_BEGIN` and `BLOCK_END`.

A `BLOCK_BEGIN` pushes the previous scope's layout list onto a scope stack so that when we reach a `BLOCK_END` we can compare that previous stack layout with how it looks after the new scope finishes. That way we know which variables were introduced by that scope and which already existed and can deallocate the local variables appropriately. 

Wherever a return statement is located, it needs to deallocate all local variables of the function, not just those declared in the scope where the return statement was found. For that reason, we keep track of all variables that been allocated since the beginning of the function. Specifically, a return statement comes bundled with a function epilogue which deallocates all local variables that have been allocated thus far and restores the callee-saved registers.

Currently, the return epilogue doesn't restore previous scopes in the scope tracking system, we just let the block initiators and terminators deal with that responsibility, although that does mean we have somewhat unoptimised code: an unreachable block deallocation is still emitted even though a preceding return jump guarantees that the extra deallocations aren't run. Probably an optimisation opportunity in there when things feel a bit more stable.

There are also operation IR instructions, for example an add operation might look like this:

`ADD_OP #3,#1,#2`

This operation implicitly declares `#3` as an in-use "virtual register", and `#1` and `#2` are the first and second operands respectively. This is what expressions flatten out to, so that every operation is a binary/unary operation with at most two sources and a single destination.

There are also `LABEL`s, which are basically the same as assembly labels. We can `JUMP_TO_ADDRESS`, given some label that we placed anywhere and don't need to worry about whether it needs range extension. The choice between range-extended jumps and standard jumps is deferred to the backend. The backend defaults to a range-extended jump and then runs a late-backend optimisation phase that finds jumps that can be reduced to short jumps using a fixed-point algorithm.

`BRANCH_IF_POSITIVE` and `BRANCH_IF_NOT_POSITIVE` are at risk of range-issues, so we still need to structure the if-statements with that in mind. Namely, we want these to jump to nearby "trampolines" (or conversely, jump over trampolines) that may or may not be long-jumps. In this case, trampolines are just `JUMP_TO_ADDRESS`.

There are function call operations in the IR, `FUNCTION_CALL` and `FUNCTION_CALL_INDIRECT` which are for normal function call and function pointer calls, respectively. For a `FUNCTION_CALL`, operand1 is the function name, and operand2 is all of the temporary variables that stand in for their respective parameters. These variable names are encoded into a string as a comma-separated list, and then unpacked in the backend. These are the only exception to the TAC convention. The `FUNCTION_CALL_INDIRECT` operation is similar, except for the fact that operand1 is not the function name, it is a temporary variable storing the function address, which was obtained via expression evaluation.

Assignment assumes that the left-hand-side is an L-value and so the expression AST-to-IR converter switches to the "addressOf" mode and thus returns the address to be assigned to. For example, take this expression

```
result = 2;
```
translates to
```
LOAD_VALUE    #1, 2
LOAD_ADDRESS  #2, result
ASSIGN_OP     #2, #1
```
The destination is `result` and `#2` is a virtual register that is set to the address of `result`. Operand 1 on the other hand, holds the R-value to be copied into the address of `result`. `ASSIGN_OP` takes an address of a memory slot in the stack as operand 1 and the value to be stored as operand 2.

The current IR is close to SSA-form like LLVM, except for the fact that it disallows virtual registers to have live ranges that span multiple basic blocks. In other words, any given virtual register should have a live range that is confined to this IR's equivalent of a basic block. The current IR generator does respect this and goes further: the live range of any virtual register is no larger than the range of the expression calculation it is part of.

This canonical form, is well-suited to the highly constrained LC-3 architecture. Seemingly, LLVM backends will sometimes mutate the IR to a form that is suitable for the targeted backend. This is after having transited forms that are more backend-agnostic. In this case, the frontend directly generates the backend friendly form and does not yet have phases transiting through backend-agnostic forms yet.

## LC-3 Backend

The LC-3 backend converts the Intermediate Representation to assembly, with some incomplete placeholders that need to be filled in based on the largely complete assembly layout. Each IR instruction is converted to some assembly, though there is some global information that alters precisely how certain IR instructions are converted. 

For example, there is the `ALLOCATE_STACK_SPACE` instruction which does not require explicit deallocation at the end of the current scope. Instead, the backend keeps track of which variable stack slots have been allocated per-function per-scope. Keeping track of allocated space on a per-scope basis allows us to determine which variables need to be deallocated at the end of each scope.

There is a late-backend optimisation system that chooses between range-extended jumps and short jumps, using a fixed-point algorithm that keeps running branch-relaxation passes until a fixed-point is reached. More details of the format of range-extended and short jumps can be found later on.

The backend has passes that determine the absolute addresses of function labels and other labels if they need to be registered in the globals table. These manually count out the number of instructions from the `.ORIG` directive, which is a responsibility of modern assemblers. This pass requires essentially the final assembly layout to be completed in order to accurately count address locations. As such, missing information in the globals table is given a placeholder designed to take up one line, just like the final layout. Therefore, filling the placeholder should not change the line count.

The following list describes the register usage conventions. Currently, the compiler doesn't have a system to track live-ranges, which means that some guarantees are made by careful use of the registers in LC-3 templates.

- `R0`
  - `R0` is used as an accumulator for calculations and intermediate results. 
  - It is also used to return single-word results directly. 
  - Another use is to accumulate pointer offsets and occasionally in pointer de-referencing.
- `R1 and R2`
  - Used to store operands for binary or unary calculations. 
  - `R1` is also used for storing pointer addresses temporarily during de-referencing assignments. 
  - callee-saved
- `R3`
  - Used for temporary additional storage to avoid clobbering, like during parameter and variable popping. 
  - Sometimes used as a variable in some implicit routines. 
  - For example, the implicit routine used for copying non-overlapping structs. 
  - callee-saved
- `R4`
  - This is a frame pointer, from which function-local parameter and variable access is indexed from. 
  - caller-saved
- `R5`
  - Used as a global symbol table pointer. 
  - Points to the beginning of the symbol table. 
  - Needs to be saved and restored if used in local internal subroutines, like the "CopyNWords" routine. 
  - global
- `R6`
  - Stack pointer. 
  - global
- `R7`
  - LC-3 default return address register. 
  - Also used as a temp store for the frame pointer, when it is available without risk of clobbering. 
  - caller-saved

Assembly labels need to be emitted for control structures, but need to make them unique to each structure. At the moment we give each control structure an index (starting from 0) based on its order of appearance in the code fragment and append it to the base label.

### Expression Evaluation

Expressions are emitted from the tree depth-first, left-to-right.

Expression ASTs are converted into the Intermediate Representation (IR) first. The IR features Three-Address-Code (TAC) style operations with "virtual registers" (represented with the prefix `#` and a number). The AST to IR conversion first flattens out nested expression trees into a linear representation where each step of the calculation is represented explicitly on its own line. 

Each line is an operation, takes in two virtual registers and places the result in another virtual register, which will likely be used in subsequent steps. The IR also has a concept of stack variables, which are stored in the stack. Future optimisation passes may move stack variables into virtual registers where possible, but this requires the fairly heavy infrastructure of CFGs and "phi" nodes which the project is not quite ready for. For example, the expression 1 + 2 + 3 is converted to 

```
LOAD_VALUE  #1, 1
LOAD_VALUE  #2, 2
ADD_OP      #3, #1, #2
LOAD_VALUE  #4, 3
ADD_OP      #5, #3, #4
```

In the LC-3 backend, virtual registers are allocated transiently in the stack's scratch space, `R0`, `R1`, `R2` and `R3`. The backend reads through the IR and accumulates assembly fragments in a map, stored in correspondence with the name of each virtual register. Each assembly fragment is precisely the series of calculations required to produce the value that will be "placed" in the corresponding virtual register.

Those calculations first compute the value, with the result landing in `R0` if a single-word value, then this is pushed onto the stack temporarily. If its a struct, then it will be pushed on top of the stack directly. In other words, each virtual register is essentially acting as a proxy for these temporary values which are stored on top of the stack.

When the backend needs those values again, they are popped into either `R0`, `R1`, `R2` or copied into another struct location, depending on which is appropriate. Some facts about expression evaluation:

- Structs are evaluated by copying their value onto the top of the stack. Further actions like assignment have to copy from the top of the stack.
- Functions returning a struct evaluate in a similar way, by placing the returned struct onto the top of the stack.

Given the very limited amount of scratchpad registers in LC-3, my suspicion is that function-scoped register allocation would have diminished returns compared to the benefits of simply adding an expression-local register allocator. For example, storing temporary values from an assignment calculation in `R0`-`R3` will reduce memory traffic.

### Function Calling Convention

`R6` is the stack pointer and this needs to be maintained across function calls, in other words, it has global scope. The chosen stack pointer invariant is that after completing a "push" or "pop" it always points one location below the top of the stack (because the stack grows downwards).

`R0` is used to return single word values. Single-word values can either be `int`s, pointers or function pointers. 

If a function returns a struct-type, a return statement will evaluate the return expression which places a copy of the struct on top of the stack. After this, the return statement will copy that struct to the struct return space allocated at the beginning of the function call and deallocate the temporary space used on top of the stack. This is possible because the address of the struct return space is pushed onto the stack by the function call, just above the new stack frame.

`R4` is the frame pointer. This needs to maintain its value for all relevant portions of a function's scope. The specific value it takes is, of course, function-local. When calling another function it will change its value, but the current value is saved by the caller before passing control to the called function and restored after the call. `R7` is set to equal the stack pointer just before function parameters are pushed on the stack and then transferred to `R4` once the parameters have been evaluated. This is because the evaluation of function parameter expressions still needs access to the caller's variables relative to the frame pointer in `R4`. Also because we want `R4` to point to the beginning of the function parameter segment of the stack.

`R4` points to the first word of the first function parameter. Function prologues and epilogues save and restore callee-saved registers. Related to this, note that the stack frame from `R4`'s location downwards has the following layout

| Stack Frame |
| :-----: |
| function-parameters |
| callee-saved-registers |
| local-variables |

When emitting code for parameter and variable access, it needs to account for the callee-saved register chunk. The function call machinery also pushes pre-defined return space on the stack, if the return type is a struct. The function call setup looks like this

- Put stack pointer in R0 
  - equals the start address of the struct return space
- Push pre-assigned struct return space 
  - only for struct return values
- Push caller-saved registers
- Push start address of the struct return space
- Set R7 to equal R6
- Push parameters
- Set R4 to equal R7
- Jump to function
- Pop parameters
- Pop starting address of the struct return space
- Restore caller-saved registers

It is important to pre-assign the struct return space (used to return the struct to the caller) before the function call. This ensures that the function's stack frame is always mutually exclusive to the struct return location. The general semantics for a function returning a struct is that it begins with a stack, and upon completion you get the same stack, plus the struct result pushed on top.

The callee-saved registers are saved within the function itself, rather than in the function call assembly.

### Convention for Loading Constants and Jump Addresses 

Using `LD` for loading constants, `JSR` for function calling and `BR` for relative jumps have limited ranges over which they can reference labelled memory locations. They use pc-offsets for referencing a memory location. In each case some small subset of the 16-bit instruction was dedicated to this. 

As a result, function jumps and label jumps have a very limited range if using those instructions. The solution to this is a symbol-table for constants and label addresses (including function labels), referenced by a pointer loaded into `R5` when the program begins, which points to the first element in this contiguous table.

In order to accommodate this, the code generation phase has a pre-processing step that calculates which address each label will correspond to, based on where the program segment starts and by counting the instructions that occur before each label. This requires all label addresses which will have entry in the symbol table to be declared in the assembly, but we don't yet know the values yet. So we create all necessary `.FILL` instructions, but with a unique placeholder for the value of each `.FILL`, which we then fill in later with the computed addresses.

Every symbol address we `.FILL` in the table will be given a label prefixed with `LITERAL-` so we can identify and count them towards the number of memory locations from the beginning of the program to a given label.

### Jump Conventions

As per the last section, this is how jumps are handled:

- When jumping to a function:
  - if a range-extended jump is required, load the function label address from the symbol table, place it in `R0` then use `JSRR R0` to jump to function
  - otherwise use a short jump: `JSR <functionLabel>`
- When returning from a function:
  - Use `JMP R7`
- When jumping to labels related to `if`, `else`, `while`:
  - if a range-extended jump is required, load the relevant label address from the symbol table, place it in `R0` then use `JMP R0` to jump to the relevant block
  - otherwise use a short jump: `BR <label>`

There isn't a way to do conditional long jumps, so we need to make use of trampoline segments. For instance, we use `BRp` to jump over an else-trampoline to the beginning of an if-clause. If the else case is triggered then we step into the trampoline which then jumps past the if clause to the else clause. This unconditional trampoline is what can either be a short or long jump. Obviously, this means we need to be careful about where we place these trampolines so they are within range of a `BRp` or similar.

### Concrete Semantics of Structs

- All structs are placed in descending order in memory. This is for ease of struct field access for stack-allocated structs. The heap is also structured in descending order so that field access can use the same logic when accessing heap-allocated structs.
- When a struct is evaluated, it is copied onto the top of the stack
- When assigning to a struct, it is copied from the top of the stack into the destination variable
- Function calls assign struct space on the stack in preparation for returning a struct. It is the first space to be pushed in a function call. Only functions that return a struct need to do this
- Return statements that are evaluating a struct put the struct value on the top of the stack and then copy it over to the pre-allocated struct return space

### Offset Handling

We still need to access variable locations, struct fields and symbol table locations using instructions with pc-offsets. Since these are rather limited in allowable size, we sometimes need to break large offsets up into a sequence of subtractions (stack is in descending order). 

Currently any offset greater than 16 in the negative direction or 15 in the positive direction is broken up into a series of subtractions from an address stored in `R0`. 

Specifically, an offset is broken up into chunks of 16 when offsetting in the negative direction and chunks of 15 in the positive. If there is a remainder then that is also subtracted.

## Supported Constructs

- Variable declarations e.g.
  - `number : int;`
  - `structItem : structType;`
  - Can modify types with `@` to declare a pointer to that type
  - Can also modify a type to create a fixed stack-array like so
    - `numberArray : int{4};`
    - `structArray : structType{6};`
    - Note that array types behave similarly to C stack-arrays, in particular, they cannot be passed by value to a function nor returned by value from a function
    - Interestingly, the reason this isn't supported is that it requires context-dependent evaluation semantics. If an array is passed as a function parameter to a function expecting an array type, then the array would need to be copied like a struct. However, if the function is expecting a pointer then a passed array should obey the array decay semantics found in C (evaluate to a pointer to the first element). In many other cases the array is evaluated as a pointer to its first element. Perhaps this is why C doesn't support it either?
    - Does not yet support multi-dimensional array syntax
- Assignment statements e.g. 
  - `number = 10;`
  - `structItem = otherStruct;`
  - Assignment statements support struct assignment, which works by copying the whole struct evaluated on the RHS into the variable on the LHS
- While loops, nestable in the standard way
  - Supports `break` statements
  - Supports `continue` statements
- If-else statements, nestable in the standard way
- User-definable structs
  - Supports nesting of structs
  - Supports pointer fields, including pointers to structs
- Pointers
  - De-referencing supported for structs and `int`s
  - Address-of operator supported for structs and `int`s
  - Supports multiple levels of pointer nesting, e.g. `var : @@int;`
- Array/pointer array access e.g. 
  - `pointer[3]`
- Complex, nested struct member access e.g.
  - `struct1.struct2->struct3.num`
  - `struct1.struct2->struct3->item`
  - `struct1.struct2->struct3.pointer[4]`
  - `struct1.struct2->struct3.pointer[4].otherItem`
  - `functionReturningStruct().third`
  - etc.
- Expressions
  - Evaluation supported in
    - RHS of assignment
    - Within function parameters
    - Within if-conditions and while-conditions
    - Within return expressions
  - Supports function calls
  - Supports most C-like operators
    - Logical
      - `&&`, `||`, `!`
      - The first two are not short-circuiting, unlike C
    - Comparison
      - `==`, `!=`, `<`, `>`, `<=`, `>=`
    - Two-Place Arithmetic
      - `+`, `-`, `*`, `/`, `%`
    - One-Place Arithmetic
      - Unary minus `-`
    - Address-of: `&`
    - Dereference: `@`
    - Struct member access
      - Direct: `.`
      - Indirect: `->`
- Functions
  - Supports functions with any number of parameters each of any type (structs included)
  - Supports functions with no parameters
  - All possible return types supported, including structs, void return-type etc.
- Function pointer types, for example:
  - `funcPointer : int[int];`
  - `funcPointer : int[structType];`
  - `funcPointer : structType[int];`
  - `funcPointer : structType[structType];`
  - `funcPointer : structType[@structType];`
  - `funcPointer : @structType[@structType];`
- Heap allocation
  - Allocate memory with `halloc` passing in desired allocation size
  - Free allocated memory by calling `free` on the pointer to the previously allocated chunk
  - `halloc` aborts if it can't find free memory
- `abort` keyword
  - Aborts the program with the message "Aborted" when used
  - Usage as a statement: `abort;`

## Compromises in Scope Relative to C

Currently, it can be seen from the test examples and the section above, that this compiler is targeting a core subset of C-like features. Albeit, with slight syntactical differences. However, some scope reductions have been chosen which might be a surprise if coming directly from C.

- Function pointers don't support immediate declaration nesting
  - Compiler doesn't support a function pointer where either the return type or parameter type is another function pointer type
  - Doesn't support, for example, `funcPointer : example[][int, int]` or `funcPointer : int[example[int], example1[int]]`
  - We can work around this by creating custom structs that contain the desired nested function pointers

The following is an example of the workaround for function pointers to achieve something like `funcPointer : int[example[int]]`

```
struct example
{
    item : int;
}

struct funcPointerTakesIntReturnsExample
{
    nestedFuncPointer : example[int];
}

function main : int ()
{
    funcPointer : int[funcPointerTakesIntReturnsExample];
    funcPointer = &toPointTo;
    return 0;
}

function toPointTo : int (funcPointerStruct : funcPointerTakesIntReturnsExample)
{
    item : example;
    item = funcPointerStruct.nestedFuncPointer(7);
    return item.item;
}
```

The choice to not support direct function pointer type nesting is essentially due to the fact that such nesting is fairly uncommon in the author's experience and often discouraged in style suggestions. However, once the current functionality is well-tested and undergone a sufficient amount of bugfixing, then this compromise can be reassessed and the direct nesting functionality could potentially be added in future versions. Additionally, the type-system was a bit ad-hoc and messy. Now that it has been tidied up and unified, expanding it to support this nesting might be more straightforward.

## Running Tests

If running from windows with WSL installed, you can run all unit tests with this command:

```
go test ./... -v
```

To run all the end-to-end tests, we have to pass over to WSL temporarily (note: GCC needs to be installed in the WSL installation). First we generate all compiled examples:

```
go run .
```

Then we run them against the virtual machine:

```
wsl ./testCompiledExamples.sh
```

The results of that run are stored in `LC-3VirtualMachine/testResult.txt`, indicating `PASS` or `FAIL` for each compiled example. `LC-3VirtualMachine/testSummary.txt` gives a summary count of the total number of tests, and the numbers of passed and failed tests. 

While the current setup is catering to windows with a WSL installation, the two prior commands are subsumed into the script:

```
.\testCompiledExamples.ps1
```

We make use of a pre-built LC-3 assembler binary, `lc3as`, which was built from a slightly modified version of this repo: [falvarezb/lc3asm](https://github.com/falvarezb/lc3asm). This binary might need to be rebuilt for your environment.

## Debugging

So far there isn't a clean and easy way to debug, aside from using useful tools such as a simulator found at [WebLC3](https://lc3.cs.umanitoba.ca/). This simulator allows you to step through the program and set breakpoints, whilst displaying register and memory state.

## LC-3 Virtual Machine Details

The VM implements trap routines using the C code, therefore, the segment of memory usually dedicated to pre-loaded LC-3 routines is free for use within the virtual machine. This trap routine segment would usually be somewhere in the region of memory from location 0 to location 16384. Making use of this fact obviously causes problems for proper compatibility but any future evolution of this project is likely to involve using a different assembly language or a modification to LC-3 to 32-bits. Therefore, bona-fide LC-3 compatibility is not too important here.

The virtual machine was written by closely following this [tutorial](https://www.jmeiners.com/lc3-vm/) by [Justin Meiners](https://www.jmeiners.com/) and [Ryan Pendleton](https://www.ryanp.me/). 

## Heap Allocation Details

The heap is based on a linked-list implementation. I chose to dedicate a fixed memory range for the heap: locations 0 to 16383. Each node/block in the linked list will be of variable size and be structured as follows
- Each block will have a three-word header
- First header word is the `free` boolean, which indicates if the block is free
  - 1 for "free", 0 for "in-use"
- Second header word is the `size` of the block (excluding these three header words)
- Third header word is a pointer to the `next` block
- The remaining chunk is the memory that can be allocated if the block indicates it is free and the size is appropriate

The list is set up so that the last block's next pointer points to `null`. Currently, the allocation algorithm will use the first free block large enough to fit the desired allocation size (first-fit heuristic). If the excess space is enough to fit a three word header and an additional word of allocatable space, then the allocator will split up the block appropriately. Specifically, it will truncate the current block to precisely fit the desired allocation space, the excess will be formed into a new block, new block's next pointer set to the current's next pointer and current's next pointer set to the new block's address.

The free function will first look for the appropriate block, whose pointer should be equal to `pointerToAllocatedMemory + 3`, then mark it as free. The offset of `3` is due to the fact that we return a pointer to the allocatable space, which is `blockPointer - 3` so as not to include the block header. After this, it will attempt to coalesce neighbouring free blocks into the largest contiguous free blocks possible. We do this because it maximises the chance we can find a block that fits our desired allocation size and can be split into a precisely fitting block and a smaller free block.

Currently, the address of the first word of the program segment, location `16384`, is being used as the `null` address. If the arrangement of segments changes, then this will need to be reconsidered. Location `16383` is used as the first word of the heap's head block and this is an invariant. The last address of the heap is location `0`, so words of the heap are arranged in reverse order, so that indexed accessing works as it does when accessing stack memory.

## Not Yet Implemented

- Syntax sugar
  - No combined declaration-assignments e.g. `value : int = 5;`
  - No increment/decrement operators e.g. `++` and `--`
  - No for loops
- Semantic analysis phase
  - Some of this has been implemented, but there is potential to make it tighter and fine-grained
  - Currently allow implicit pointer casting to an extent, as locking this down more requires syntax for explicit casting and ideally that might deviate from the C style to make it easier to parse.
- Better syntactic error reporting (In progress)
  - Currently, there is basic syntax error reporting, which occurs when `consumeToken` encounters an unexpected token.
- `&&` and `||` could be made to be short-circuiting as is standard
  - Within the current canonical IR form, this could be implemented by using a hidden stack allocated variable to facilitate mutations across branching code (since short-circuiting requires control flow)
- Arrays currently don't support multi-dimensional declarations as far as I'm aware i.e. `array : int{5}{5};`
  - Can achieve something similar by heap allocating to a double pointer.
  - Another workaround would be calculating the total entry count and arranging the array in row-major form.

## Execution Semantics Tests That Try to Detect Memory Corruption

Here I'll note down some of the strange end-to-end tests, particularly if they rely too heavily on implementation-specific details and might be better having a unit test instead.

Most of these odd tests, and some not mentioned here yet, are trying to detect memory corruption in a way that may not be portable, although I'm at liberty to define the standard and I haven't decided for sure yet. Some of the other tests use a sentinel value or array, which is allocated in a while loop of many iterations. The while loop performs some calculation for a result repeatedly, but its idempotent: its just recalculating the same value. The sentinel value is set on the first iteration only and it should leave a ghost value at some defined distance away from the stack pointer location after while-loop completion. 

By allocating a similar array just after the while loop, we can capture the ghost of the sentinel value on the stack and if its the same as what it was set to in the first iteration, then there was likely no stack drift. Of course, this is assuming that variables are allocated locally to each scope block, but pre-allocating all variables in the backend is a valid strategy, which I believe LLVM uses. In this case, scoping rules are enforced by the frontend and seemingly not present in the backend.

## Optimisation passes

There are a small number of optimisations, having migrated to the IR will make it easier to add more. The first optimisation I had was making struct access largely work in-place with structs that are already in the stack frame, with the exception of R-value structs returned from a function. By "in-place" I mean that the struct isn't copied anywhere to work with it. Even with the R-value returned structs, the only additional copy is the one the return handling uses to copy it to the designated return slot on the stack.

Then, there is a peephole optimisation to remove redundant, neighbouring push and pop operations.

There is a pre-IR optimisation phase that operates on the AST. This is constant folding, which reduces reducible expressions as much as possible. For example, if we write `result = 1 + 2;` then the AST is reduced to the AST that would have resulted from `result = 3;`. Obviously, this leaves irreducible expressions, usually those which have a variable or function call as one of the operands. This essentially requires identifying minimal subexpressions patterns that can then be rewritten to either pre-compute operations on constants, or open up more opportunities to reduce constants.

There is a late-backend phase which converts the default range-extended jumps to short jumps if the target label is in range. This is done with a fixed-point algorithm as each conversion opens up opportunities for previous jumps that were initially too far away. Since this optimisation removes instructions, phases that work by counting instruction locations have to run after this optimisation phase.