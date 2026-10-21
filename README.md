# C-style Language Compiler Targeting LC-3

## Lexer

The lexer is configured with a priority-ordered list of regexes, each designed to match the corresponding token substring. The algorithm matches from the beginning of the string with all the token regexes, and when we find a match, we read off a token and add it to the accumulating token array.

The lexer sometimes will need to disambiguate when multiple token regexes find a match. It does this by finding the joint longest matches, and picking the unique joint longest match with the highest priority.

The selected token substring is removed from the beginning of the byte stream, so that the process is ready for the next token to be read.

Currently, the lexer is maximising readability and extensibility. However, to make it more efficient there are some tactics that will likely help lower its runtime cost.

- Stop using regexes for single-character tokens and just check for them directly in some switch-case statement
- If none of those trigger, then check if the current character is alphabetic
  - If the current character is alphabetic then we can fall back on the token regex list approach, but the only token types in the list will be alphabetic tokens e.g. name tokens and keyword tokens
  - Or just greedily consume characters that might be part of a name or keyword. Once we reach a character that can't be part of a name or keyword, then we look back at the accumulated characters and try to discern which token it was.
- Then check for numeric characters. Currently no name token or keyword token is allowed to begin with a numeric character
  - In this case we could greedily consume numeric characters into an accumulating token, stopping when the current character is not numeric

## Parser

The parser is a hand-written recursive descent parser. The expression sector of the parser uses a variant of recursive descent with adjustments to handle infix operator precedence. I didn't use Pratt parsing, because it would have been too easy to just copy the algorithm and gain no further insight. The alternative is muddling through and feeling stupid for a while, but often that is when you learn the most. Ironically, after all that exploratory stumbling, I was able to look at Pratt parsing with a clearer idea of the specific tree structures it is cleverly and economically building up.

The parser I had during the prototype stage was actually table-based. This was a hold over from a previous project where I was exploring how parsing works and partly wanted to figure out how parser-generators work. In that project I also ended up writing a basic regular expression engine, some generic data structures and explored how to implement a form of OOP in C. That was likely part of the reason I included function pointers in this language.

## Intermediate Representation

The intermediate representation is largely an abstracted/generic assembly language with extra utilities like function calls and struct handling operations.

The IR allows you to declare stack variable allocations with `ALLOCATE_STACK_SPACE`. For current purposes, these are guaranteed to be translated into stack variables, which seems to be a detail that LLVM exposes (`alloca`). There are also "virtual registers", which are used in operations that take Three-Address-Code form. These "registers" are numbers prefixed with `#` since the lexer and parser don't allow that character in variable names ensuring disambiguation with every possible user variable.

The local variables declared on the stack don't need to be explicitly deallocated. The lifetimes of these variables are managed by the backend with reference to the scope-block markers: `BLOCK_BEGIN` and `BLOCK_END`.

A `BLOCK_BEGIN` pushes a new scope onto a scope stack and new variable declarations in the new scope are added to the scope on top of the stack. Having kept track of which variables were introduced by the current scope, it is only the memory associated with those variables that needs to be deallocated when a `BLOCK_END` is encountered. 

This differs from the deallocation behaviour of `return` statements. Returns need to deallocate all local variables introduced in the function up till that point. A `return` statement comes bundled with a function epilogue that performs the necessary deallocations.

Currently, the `return` epilogue doesn't restore previous scopes in the scope tracking system, the block initiators and terminators deal with that responsibility, although that does mean there is some unreachable, unoptimised code: a block deallocation is still emitted even though a preceding return jump guarantees that the extra deallocations aren't run.

There are also binary/unary operations in the IR, for example, an add operation:

`ADD_OP #3,#1,#2`

This operation implicitly declares `#3` as an in-use "virtual register", and `#1` and `#2` are the first and second operands respectively. The AST-to-IR conversion flattens expressions out so that most operations are a binary/unary operation with at most two sources and a single destination.

There are also `LABEL`s, which serve the same purpose as labels in typical assembly languages. We can `JUMP_TO_ADDRESS`, given some label that we placed anywhere and don't need to worry about whether it needs range extension. The choice between range-extended jumps and standard jumps is deferred to a late-backend phase. The backend defaults to a range-extended jump and then runs a late-backend fixed-point optimisation algorithm that reduces range-extended jumps to short jumps.

`BRANCH_IF_POSITIVE` and `BRANCH_IF_NOT_POSITIVE` are at risk of range-issues, so we still need to structure the if-statements with that in mind. Namely, we want these to jump to nearby "trampolines" (or conversely, jump over trampolines) that may or may not be long-jumps. In this case, trampolines are just `JUMP_TO_ADDRESS`.

There are IR operations that subsume the operations associated with function calls. `FUNCTION_CALL` and `FUNCTION_CALL_INDIRECT` which are for normal function calls and indirect function calls, respectively. For a `FUNCTION_CALL`, operand1 is the function name, and the remaining operands are a list of all the temporary variables that stand in for their respective parameters. The function call instructions are the only exception to the TAC convention. The `FUNCTION_CALL_INDIRECT` operation is similar, except for the fact that operand1 is not the function name, it is a temporary variable storing the function address, which was computed via an expression.

Assignment requires the left-hand-side to be a valid lvalue. The expression sector of the AST-to-IR converter switches over to the "addressOf" mode, thereby returning the address to assign the RHS' value to. For example, take this expression

```
result = 2;
```
translates to
```
LOAD_VALUE    #1, 2
LOAD_ADDRESS  #2, result
ASSIGN_OP     #2, #1
```
The destination is `result` and `#2` is a virtual register that is set to the address of `result`. Operand 1, on the other hand, holds the R-value to be copied into the address of `result`. `ASSIGN_OP` takes the address of a memory slot in the stack as operand 1 and the value to be stored at the address as operand 2.

A `COPY_STRUCT` instruction was added to facilitate direct struct copying from an lvalue struct to an lvalue struct during assignment. Previously, a struct expression on the RHS was evaluated by copying the struct to the top of the stack and then final operation of the assignment would copy the struct from the top of the stack to the destination. In order to do the direct struct copy, it seems as though the IR needs to expose some of this information rather than abstracting it.

The current IR conforms to an SSA-like layout of virtual registers in expressions but only within expressions and/or basic blocks. In other words, any given virtual register should have a live range that is confined to this IR's equivalent of a basic block. The current IR generator does respect this and goes further: the live range of any virtual register is no larger than the range of the expression calculation it is part of.

This canonical form, is well-suited to the highly constrained LC-3 architecture. In this case, the frontend directly generates the backend friendly form and does not yet have optimisation phases that transit through backend-agnostic forms.

## LC-3 Backend

The LC-3 backend converts the Intermediate Representation to LC-3 assembly. The initial translation adds some incomplete placeholders whose final values are based on an assembly layout that is largely reflective of the final layout. There is some global information that alters precisely how certain IR instructions are converted. 

The `ALLOCATE_STACK_SPACE` instruction does not require explicit deallocation at the end of the current scope. Instead, the backend keeps track of which variable stack slots have been allocated per-function per-scope. Keeping track of allocated space on a per-scope basis allows the backend to determine which variables need to be deallocated at the end of each scope.

There is a late-backend optimisation system that chooses between range-extended jumps and short jumps, using a fixed-point algorithm that keeps running branch-relaxation passes until a fixed-point is reached. More details of the format of range-extended and short jumps can be found later on.

The backend has an assembly finalisation pass (somewhat like parts of a linker's responsibility) that determines the absolute addresses of function labels and other labels if they need to be registered in the globals table. This pass manually counts out the number of instructions from the `.ORIG` directive, requiring the assembly layout to correspond with the final layout in order for the line counting to be accurate. Therefore, missing information in the globals table is given a placeholders which take up the same number of lines as the final layout.

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

Assembly labels must be unique for control structures to avoid naming clashes. Each control structure (`if-else` statements and `while` loops) is given a unique, sequential ID (starting from 0). When these control structures are converted into IR, their labels are constructed with their IDs to disambiguate them.

### Expression Evaluation

Expressions are emitted from the tree depth-first, left-to-right.

Expression ASTs are converted into the Intermediate Representation (IR) first. The IR is based around a Three-Address-Code (TAC) style with "virtual registers" (represented with the prefix `#` and a number). The AST-to-IR conversion flattens out nested expression trees into a linear representation where each step of the calculation is explicitly represented on its own line. 

Each operation takes in two virtual registers and places the result in another virtual register which may itself become an operand for a subsequent operation. The IR also has a concept of stack variables, which are stored in the stack. Future optimisation passes may move stack variables into virtual registers where possible, but this requires the fairly heavy infrastructure of CFGs and "phi" nodes which the project is not quite ready for. For example, the expression 1 + 2 + 3 is converted to 

```
LOAD_VALUE  #1, 1
LOAD_VALUE  #2, 2
ADD_OP      #3, #1, #2
LOAD_VALUE  #4, 3
ADD_OP      #5, #3, #4
```

In the LC-3 backend, virtual registers are allocated transiently in the stack's scratch space, `R0`, `R1`, `R2` and `R3`. The backend reads through the IR and accumulates assembly fragments in a map, stored in correspondence with the name of each virtual register. Each assembly fragment is precisely the series of calculations required to produce the value that will be "placed" in the corresponding virtual register.

Those calculations first compute the value, with the result landing in `R0` if a single-word value, then this is pushed onto the stack temporarily. If it's a struct, then it will be pushed on top of the stack directly. In other words, each virtual register is essentially acting as a proxy for these temporary values which are stored on top of the stack.

When the backend needs those values again, they are popped into either `R0`, `R1`, `R2` or copied into another struct location, depending on which is appropriate.

Given the very limited amount of scratchpad registers in LC-3, my suspicion is that function-scoped register allocation would have diminished returns compared to the benefits of simply adding an expression-local register allocator. For example, storing temporary values from an assignment calculation in `R0`-`R3` will reduce memory traffic.

### Function Calling Convention

`R6` is the stack pointer and this needs to be maintained across function calls, in other words, it has global scope. The chosen stack pointer invariant is that after completing a "push" or "pop" it always points one location below the top of the stack (because the stack grows downwards).

`R0` is used to return single-word values: `int`s, pointers or function pointers. 

If a function returns a struct type, a return statement evaluates the return expression which places a copy of the struct on top of the stack. After this, the return handling copies that struct to the pre-allocated struct return space and deallocates the temporary space on top of the stack. This process uses the address of the struct return space which was pushed onto the stack by the caller.

`R4` is the frame pointer. The specific value it takes is function-local. The value it has for the caller function is saved by the caller before passing control to the callee function and restored after the call. After `R7` is saved by the caller it is used to store the callee's frame pointer, which equals the stack pointer before parameters are pushed onto the stack. That value is transferred to `R4` once the parameters have been evaluated because the function parameter expression evaluation still needs access to the caller's variables.

`R4` points to the first word of the first function parameter. Function prologues and epilogues save and restore callee-saved registers. Related to this, note that the stack frame from `R4`'s location downwards has the following layout

| Stack Frame |
| :-----: |
| function-parameters |
| callee-saved-registers |
| local-variables |

When emitting code for parameter and variable access, the callee-saved register chunk needs to be accounted for by adding an additional offset value once we pass from the parameter section to the local variable section. The function call machinery also pushes pre-defined return space on the stack, if the return type is a struct. The function call setup looks like this

- Put stack pointer in R0 
  - Is the address of the struct return space
- Allocate struct return space 
  - Only if the function returns a struct
- Push caller-saved registers
- Push the address of the struct return space
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

There are some limitations in LC-3's instruction set to be aware of. These are evident when using `LD` for loading constants, `JSR` for function calling and `BR` for relative jumps. They use PC-offsets to reference a labelled memory location. In each case some small subset of the 16-bit instruction is dedicated to a PC-offset. This imposes limits on the range over which constants can be retrieved, conditional blocks jumped over, and subroutines jumped to if these instructions were the only options available.

The solution I chose for this is a symbol-table for constants and label addresses (including function labels), referenced by a pointer loaded into `R5` when the program begins, which points to the first element in this contiguous table. To access constants and label addresses in the table, `R5` is moved into `R0`, the appropriate offset is applied, and then the value at that slot in the table is loaded with `LDR`.

In order to accommodate this, the code generation phase has a pre-processing step that calculates which address each label will correspond to, based on where the program segment starts and by counting the instructions that occur before each label. This requires all address entries in the symbol table to be present before the final address value is known. Since entries in the table map to `.FILL` instructions these are laid out before the addresses are known, with a unique placeholder for the concrete address values which are filled in later.

Every symbol address we `.FILL` in the table will be given a label prefixed with `LITERAL-` so we can identify and count them towards the number of memory locations from the beginning of the program to a given label.

One alternative approach may be to put the table after the program segment, but this would require a new approach to bootstrap `R5`. Since the `.ORIG` directive must be at the top of the program segment.

### Jump Conventions

As per the last section, this is how jumps are handled:

- When jumping to a function:
  - if a range-extended jump is required, load the function label address from the symbol table into `R0` then use `JSRR R0` to jump to function
  - otherwise use a short jump: `JSR <functionLabel>`
- When returning from a function:
  - Use `JMP R7`
- When jumping to labels related to `if`, `else`, `while`:
  - if a range-extended jump is required, load the relevant label address from the symbol table into `R0` then use `JMP R0` to jump to the relevant block
  - otherwise use a short jump: `BR <label>`

There is no way to do conditional long jumps, so we need to conditionally jump to or over unconditional trampoline segments. For instance, we use `BRp` to jump over an else-trampoline to the beginning of an if-clause. If the `else` case is triggered then we step into the trampoline that jumps over the if clause. This unconditional trampoline is what can either be a short or long jump. This means we need to be careful about where we place these trampolines so they are within range of a `BRp` or similar.

### Concrete Semantics of Structs

- All structs are placed in descending order in memory, including those in the heap. The heap is also structured in descending order so that various operations can reuse the same code-generation.
- When a struct is evaluated, it is copied onto the top of the stack.
  - Unless the struct is the RHS of assignment and both the LHS and RHS are lvalues, then the struct is copied directly to the destination.
- When assigning to a struct, it is copied from the top of the stack into the destination variable.
  - Unless both the LHS and RHS are lvalues, then the source struct is directly copied from the RHS's location.
- Functions returning a struct evaluate in a similar way, by placing the returned struct onto the top of the stack.
  - Function calls assign struct space on the stack in preparation for returning a struct. It is the first chunk of stack memory allocated in a function call.
- Return statements that evaluate a struct put the struct value on the top of the stack and then it is copied to the pre-allocated struct return space.

### Offset Handling

Despite using various tricks to avoid PC-offset limitations, PC-offsets are difficult to avoid entirely. At the moment they are still used when applying offsets to registers. Sometimes these offsets need to broken up into a sequence of subtractions or additions. 

Currently any offset greater than 16 in the negative direction or 15 in the positive direction is broken up into a series of subtractions/additions from an address stored in `R0`. 

Specifically, an offset is broken up into chunks of 16 when offsetting in the negative direction and chunks of 15 in the positive. If there is a remainder then that is also subtracted/added.

## Supported Constructs

- Variable declarations e.g.
  - `number : int;`
  - `structItem : structType;`
  - Can modify types with `@` to declare a pointer to that type
  - Can also modify a type to create a fixed stack-array like so
    - `numberArray : int{4};`
    - `structArray : structType{6};`
    - Arrays decay to pointers to the first element much like C
    - Does not yet support multi-dimensional array syntax
- Assignment statements e.g. 
  - `number = 10;`
  - `structItem = otherStruct;`
  - Assignment statements support struct assignment, which works by copying the whole struct evaluated on the RHS into the variable on the LHS
- While loops
  - Supports `break` statements
  - Supports `continue` statements
  - Can be nested in the standard way
- If-else statements
  - Including `else if` chaining
  - Can be nested in the standard way
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
    - Function pointer expressions can include struct member access
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
  - Supports any number of parameters each of any type (structs included)
  - Supports with no parameters
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

## Running Tests

Currently, the auxiliary scripts assume you are running Windows with WSL installed. It doesn't take many tweaks to switch it over to Linux. Since the execution test framework needs to build an LC-3 VM binary from a C file, GCC needs to be installed on your WSL installation. You can run all unit tests with this command:

```
go test ./... -v
```

Main currently generates all compiled examples:

```
go run .
```

To run the execution test runner against existing compiled assembly files:

```
wsl ./testCompiledExamples.sh
```

The results of the run are stored in `LC-3VirtualMachine/testResult.txt`, indicating `PASS` or `FAIL` for each compiled example. `LC-3VirtualMachine/testSummary.txt` gives a summary count of the total number of tests, and the numbers of passed and failed tests. 

While the current setup is catering to Windows with a WSL installation, the two prior commands are subsumed into the script:

```
.\testCompiledExamples.ps1
```

This process uses a pre-built LC-3 assembler binary, `lc3as`, which was built from a slightly modified version of this repo: [falvarezb/lc3asm](https://github.com/falvarezb/lc3asm). This binary might need to be rebuilt for your environment.

## Debugging

So far there isn't a debugger, turns out those are very difficult to write for a compiler. Sometimes its useful to use a simulator like this one [WebLC3](https://lc3.cs.umanitoba.ca/). This simulator allows you to step through the program and set breakpoints, whilst displaying register and memory state.

## LC-3 Virtual Machine Details

The VM implements trap routines using the C code, therefore, the segment of memory usually dedicated to pre-loaded LC-3 routines is free for use within the virtual machine. This trap routine segment would usually be somewhere in the region of memory from location 0 to location 16384. Making use of this fact obviously causes problems for proper compatibility but any future evolution of this project is likely to involve using a different assembly language or a modification to LC-3 to 32-bits. Therefore, bona-fide LC-3 compatibility is not too important here.

The virtual machine was written by closely following this [tutorial](https://www.jmeiners.com/lc3-vm/) by [Justin Meiners](https://www.jmeiners.com/) and [Ryan Pendleton](https://www.ryanp.me/). 

## Heap Allocation Details

The heap uses a contiguous linked-list implementation. I chose to dedicate a fixed memory range for the heap: locations 0 to 16383. Each node/block in the linked list will be of variable size and be structured as follows
- Each block will have a three-word header
- First header word is the `free` boolean, which indicates if the block is free
  - 1 for "free", 0 for "in-use"
- Second header word is the `size` of the block (excluding these three header words)
- Third header word is a pointer to the `next` block
- The remaining chunk is the memory that can be allocated if the block indicates it is free and the size is appropriate

The list is set up so that the last block's next pointer points to `null`. Currently, the allocation algorithm will use the first free block large enough to fit the desired allocation size (first-fit heuristic). If the excess space is enough to fit a three word header plus an additional word (or more) of allocatable space, then the allocator will split the block. It will truncate the current block to precisely fit the desired allocation space, the excess will be formed into a new block. The new block's next pointer is set to the current's next pointer and current's next pointer set to the new block's address.

The free function will first look for the appropriate block, whose pointer should be equal to `pointerToAllocatedMemory + 3`, then mark it as free. The offset of `3` is due to the fact that we return a pointer to the allocatable space, which is `blockPointer - 3` so as not to include the block header. After this, it will attempt to coalesce neighbouring free blocks into the largest contiguous free blocks possible. This maximises the chance that a subsequent allocation will find a block that can fit the desired size plus excess space for a new block (or a small amount of fragmentation).

Currently, the address of the first word of the program segment, location `16384`, is being used as the `null` address. If the arrangement of segments changes, then this will need to be reconsidered. Location `16383` is used as the first word of the heap's head block and this is an invariant. The last address of the heap is location `0`, so words of the heap are arranged in reverse order, so that indexed accessing works as it does when accessing stack memory.

## Not Yet Implemented

- Syntax sugar
  - No combined declaration-assignments e.g. `value : int = 5;`
  - No increment/decrement operators e.g. `++` and `--`
  - No for loops
- Semantic analysis phase
  - Most of this is now implemented, but perhaps not as ergonomic as something like Rust
  - Currently allow implicit pointer casting for normal pointers
    - To add explicit casting need to add syntax for it: would be better if the syntax was easier to parse than C's version
- Better syntactic error reporting
  - Currently, there is basic syntax error reporting, which occurs when `consumeToken` encounters an unexpected token
- `&&` and `||` could be made to be short-circuiting as is standard
- Currently don't support multi-dimensional array declarations i.e. `array : int{5}{5};`
  - Can achieve something similar by heap allocating to a double pointer
  - Another workaround would be calculating the total entry count and arranging the array in row-major form
- Function pointers don't support immediate declaration nesting
  - Compiler doesn't support a function pointer where either the return type or parameter type is another function pointer type
  - Doesn't support, for example, `funcPointer : example[][int, int]` or `funcPointer : int[example[int], example1[int]]`
  - We can work around this by creating custom structs that contain the desired nested function pointers
- No global variables
- No module system
- Only a single file can be compiled, however there is fledgling linking functionality which could be adapted
- Currently the ABI has been explicit that operations put a struct on the stack, before moving it on to some destination. Now that the architecture is nicer, can start to consider optimisations that directly copy the struct without transiting the top of the stack
  - Have done this for assignment statements where both the LHS and RHS are addressable lvalues
  - Still need to do this for the extra copy operation that occurs when moving a struct value to the struct return location

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

These scope reductions helped to prioritise correctness, and laying a good foundation. 

## Execution Semantics Tests That Try to Detect Memory Corruption

Some tests try to detect memory corruption in a way that may not be portable, but that really depends on how much further I want to take the language. Some of these tests use a sentinel value or array, which is allocated in a while loop of many iterations. The while loop performs some calculation for a result repeatedly, but its idempotent: its just recalculating the same value. The sentinel value is set on the first iteration only and it should leave a ghost value in the stack region that is marked as deallocated after the while loop completes. 

By allocating a similar array just after the while loop, the "ghost" of the sentinel value can be captured and if its the same as what it was set to in the first iteration, then there was likely no stack drift. This is assuming that variables are allocated locally to each scope block, although pre-allocating all variables at the beginning of a function is a valid strategy.

## Optimisation passes

The first optimisation I had (assuming that some hobbyist compilers might not do this) was making struct access largely work in-place with structs. R-value structs need to be handled as temporaries although struct member access still works in-place. "In-place" means that nested structs aren't copied onto the top of the stack to access the nested fields.

There is a peephole optimisation to remove redundant, neighbouring push and pop operations.

There is constant folding and algebraic rewriting, performed on the AST. For example, starting with `result = 1 + 2;` the AST is reduced to the AST that would have been parsed from `result = 3;`. This essentially requires identifying minimal sub-expression patterns that can then be rewritten, wither to pre-compute operations on constants, rewrite algebraic expressions to minimise particular operations, or open up more opportunities for either of those by doing tree-rewrites.

There is a late-backend phase which converts the default range-extended jumps to short jumps if the target label is in range. This is done with a fixed-point algorithm as each conversion opens up opportunities for previous jumps that were initially too far away. Since this optimisation removes instructions, phases that work by counting instruction locations have to run after this optimisation phase.