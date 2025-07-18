# Micro Compiler Implemented with Flex and Bison

Course project for CSC4180: Compiler Construction @ CUHK(SZ), 2024 Spring.

---

## Code Structure

```
project
|-
|--- README.md
|-
|--- testcases
|-
|--- src
	|-
	|--- ir_generator.cpp
	|--- ir_generator.hpp
	|--- main.cpp
	|--- Makefile
	|--- node.cpp
	|--- node.hpp
	|--- parser.y
	|--- scanner.l
```

## How to execute the compiler?

You can test the full compiler pipeline (scanner → parser → IR → RISC-V → simulation) using the provided shell script:

```bash
bash run_all_tests.sh

## How do I design the Scanner?

The scanner is designed to extract tokens of Micro language according to regular expressions. There are totally 14 tokens in Micro and the Regular Expression rules I designed are addressed below:

```
EOLN        "\n"
TAB         "\t"
COMMENT     --.*\n
BEGIN_      "begin"
END         "end"
READ        "read"
WRITE       "write"
LPAREN      "("
RPAREN      ")"
SEMICOLON   ";"
COMMA       ","
ASSIGNOP    ":="
PLUSOP      "+"
MINUSOP     "-"
ID          [a-zA-Z][a-zA-Z0-9_]{0,31}
INTLITERAL  -?[0-9]+
```

Extracting the tokens using Regular Expressions, we can get tokens stored with type `yylval.strval` or `yylval.intval`, and pass to the parser as `T_TOKENNAME`.

The scanner is also able to print both token class and lexeme (i.e. `<token-class, lexeme>`) for each token with `--scan-only` option. Here is a sample output for the print function on `test0.m`:

```
<BEGIN_, begin>
<ID, A>
<ASSIGNOP, :=>
<INTLITERAL, 10>
<SEMICOLON, ;>
<ID, B>
<ASSIGNOP, :=>
<ID, A>
<PLUSOP, +>
<INTLITERAL, 20>
<SEMICOLON, ;>
<WRITE, write>
<LPAREN, (>
<ID, B>
<RPAREN, )>
<SEMICOLON, ;>
<END, end>
<SCANEOF>
```

## How do I design the Parser?

The Parser is designed to receive the tokens extracted from scanner. It also generates a parse tree (or concrete syntax tree), and futhermore, the abstract syntax tree (AST) based on the context-free grammar (CFG).

Here is the CFG of Micro language:

```
<start> → <program> SCANEOF
<program> → BEGIN <statement list> END
<statement list> → <statement> { <statement> }
<statement> → ID ASSIGNOP <expression>;
<statement> → READ LPAREN <id list> RPAREN;
<statement> → WRITE LPAREN<expr list> RPAREN;
<id list > → ID { COMMA ID }
<expr list > → <expression> { COMMA <expression> }
<expression> → <primary> { <add op> <primary> }
<primary> → LPAREN <expression> RPAREN
<primary> → ID
<primary> → INTLITERAL
<add op> → PLUSOP
<add op> → MINUSOP
```

In `parser.y`, I designed three `yylval` data types:

```c++
%union {
	/* Data */
	int intval;
	const char* strval;
	struct Node* nodeval;
};
```

Then, assign these data types to terminal symbols and non-terminal symbols.

##### Terminal Symbols:

```
%token <intval> T_INTLITERAL
%token <strval> T_BEGIN_ T_END T_READ T_WRITE T_LPAREN T_RPAREN T_SEMICOLON 
%token <strval> T_COMMA T_ASSIGNOP T_PLUSOP T_MINUSOP T_SCANEOF T_ID
```

##### Non-Terminal Symbols:

```
%type <nodeval> program statement_list statement id_list expr_list expression primary
```

## How do I design the Intermediate Code Generator?

The intermediate representation (IR) of the compiler is the LLVM IR, and the IR generator converts the AST given from the parser to LLVM IR (`.ll`) file. The `ir_generator.cpp` file has the following structure:

##### Export AST to LLVM IR

The `export_ast_to_llvm_ir()` function generates the LLVM IR header and main function structure. And it calls the `gen_llvm_ir()` function.

##### Generate LLVM IR

The generate LLVM IR sequence recursively calls the main routine or the sub-routines to generate the full LLVM IR.

-   Function `gen_llvm_ir()`

This is the main routine in the generation part, it calls sub-routines such as `gen_assignop_llvm_ir()`, `gen_read_llvm_ir()`, `gen_write_llvm_ir()`, and also, the main routine `gen_llvm_ir()` to traverse the whole AST tree.

-   Function `gen_assignop_llvm_ir()`

This sub-routine handles the following LLVM IR instructions:

```
%<variable> = alloca i32
store i32 <rvalue>, i32* %<variable>
```

-   Function`gen_read_llvm_ir()`

This sub-routine handles the following LLVM IR instructions:

```
%<variable> = alloca i32
%_scanf_format_1 = alloca [# x i8]
store [# x i8] c"%d ... %d\00", [# x i8]* %_scanf_format_1
%_scanf_str_1 = getelementptr [# x i8], [# x i8]* %_scanf_format_1, i32 0, i32 0
call i32 (i8*, ...) @scanf(i8* %_sacnf_str_1, i32* %<variable>)
```

-   Function `gen_write_llvm_ir()`

This sub-routine handles the following LLVM IR instructions:

````
%_printf_format_1 = alloca [# x i8]
store [# x i8] c"%d ... %d\0A\00", [# x i8]* %_printf_format_1
%_printf_str_1 = getelementptr [# x i8], [# x i8]* %_printf_format_1, i32 0, i32 0
call i32 (i8*, ...) @printf(i8* %_printf_str_1, i32 <rvalue>)
````

-   Function `gen_operation_llvm_ir()`

This sub-routine handles the following LLVM IR instructions:

```
%_tmp_1 = load i32, i32* %<variable>
%_tmp_2 = sub i32 <expression>, %_tmp_1
%_tmp_3 = add i32 %_tmp_1, %_tmp_2
```

-   Function `gen_expression_llvm_ir()`

This sub-routine handles the following LLVM IR instruction:

```
%_tmp_1 = load i32, i32* %<variable>
```
