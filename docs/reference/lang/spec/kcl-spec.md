---
title: "KCL Spec"
linkTitle: "KCL Spec"
type: "docs"
weight: 2
description: KCL Spec
---

## Lexical rules

### Keywords and reserved words

The following identifiers are used as reserved words, or keywords of the language, and cannot be used as ordinary identifiers. They must be spelled exactly as written here:

```
True       False      None        Undefined   import
and        or         in          is          not
as         if         else        elif        for
schema     mixin      protocol    check       assert
all        any        map         filter      lambda
rule
```

The following tokens are not used, but they are reserved as possible future keywords:

```
pass       return     validate   rule        flow
def        del        raise      except      try
finally    while      from       with        yield
global     nonlocal   struct     class       final
```

### Line comment

```
# a comment
```

### Operators

```
+       -       *       **      /       //      %
<<      >>      &       |       ^       <       >
~       <=      >=      ==      !=      =
+=      -=      *=      **=     /=      //=     %=
<<=     >>=     &=      ^=
```

### Delimiters

```
(       )       [       ]       {       }
,       :       .       ;       @
```

### Operator precedence

The following list of operators is ordered from **highest to lowest**:

| Operator                                                                         | Description                                              |
| -------------------------------------------------------------------------------- | -------------------------------------------------------- |
| `**`                                                                             | Exponentiation (highest priority)                        |
| `+x` `-x` `~x`                                                                   | Positive, negative, bitwise NOT                          |
| `*` `/` `%` `//`                                                                 | Multiplication, division, floor division and remainder   |
| `+` `-`                                                                          | Addition and subtraction                                 |
| `<<` `>>`                                                                        | Left and right shifts                                    |
| `&`                                                                              | Bitwise AND                                              |
| `^`                                                                              | Bitwise XOR                                              |
| \|                                                                               | Bitwise OR                                               |
| `in`, `not in`, `is`, `is not`, `<`, `<=`, `>`, `>=`, `!=`, `==`                 | Comparisons, including membership and identity operators |
| `not`                                                                            | Boolean NOT                                              |
| `and`                                                                            | Boolean AND                                              |
| `or`                                                                             | Boolean OR                                               |
| `if – else`                                                                      | Conditional expression =                                 |
| `=`, `+=`, `-=`, `*=`, `/=`, `%=`, `&=`, `=`, `^=`, `\*\*=`, `//=`, `<<=`, `>>=` | Assign                                                   |

## Grammar

KCL uses the Python-based [LarkParser](https://lark-parser.readthedocs.io/en/latest/) tool to describe the grammar, and the specification rules are as follows:

```bnf
//////////// KCL grammar ////////////
start: (NEWLINE | statement)*

statement: simple_stmt | compound_stmt
simple_stmt: (assign_stmt | unification_stmt | expr_stmt | assert_stmt | import_stmt | type_alias_stmt) NEWLINE
compound_stmt: if_stmt | schema_stmt | rule_stmt

//////////// import_stmt ////////////
import_stmt: IMPORT dot_name (AS NAME)?
dot_name: (leading_dots identifier) | identifier
leading_dots: DOT+

/////////// assert_stmt ////////////
assert_stmt: ASSERT simple_expr (IF simple_expr)? (COMMA test)?

//////////// if_stmt ////////////
if_stmt: IF test COLON execution_block (ELIF test COLON execution_block)* (ELSE COLON execution_block)?
execution_block: if_simple_stmt | NEWLINE _INDENT schema_init_stmt+ _DEDENT
if_simple_stmt: (assign_stmt | unification_stmt | expr_stmt | assert_stmt) NEWLINE

//////////// assign_stmt ////////////
assign_stmt: identifier [COLON type] (ASSIGN identifier)* ASSIGN test
    | identifier (COMP_PLUS | COMP_MINUS | COMP_MULTIPLY | COMP_DOUBLE_STAR | COMP_DIVIDE
    | COMP_DOUBLE_DIVIDE | COMP_MOD | COMP_AND | COMP_OR | COMP_XOR | COMP_SHIFT_LEFT
    | COMP_SHIFT_RIGHT) test

//////////// unification_stmt ////////////
unification_stmt: identifier COLON schema_expr

//////////// schema_stmt ////////////
schema_stmt: [decorators] (SCHEMA|MIXIN|PROTOCOL) NAME [LEFT_BRACKETS [schema_arguments] RIGHT_BRACKETS] [LEFT_PARENTHESES identifier (COMMA identifier)* RIGHT_PARENTHESES] [for_host] COLON NEWLINE [schema_body]
schema_arguments: schema_argument (COMMA schema_argument)*
schema_argument: NAME [COLON type] [ASSIGN test]
schema_body: _INDENT (string NEWLINE)* [mixin_stmt] (schema_attribute_stmt|schema_init_stmt|schema_index_signature)* [check_block] _DEDENT
schema_attribute_stmt: attribute_stmt NEWLINE
attribute_stmt: [decorators] (identifier | STRING) [QUESTION] COLON type [(ASSIGN|COMP_OR) test]
schema_init_stmt: if_simple_stmt | if_stmt
schema_index_signature: LEFT_BRACKETS [NAME COLON] [ELLIPSIS] basic_type RIGHT_BRACKETS COLON type [ASSIGN test] NEWLINE

//////////// rule_stmt ////////////
rule_stmt: [decorators] RULE NAME [LEFT_BRACKETS [schema_arguments] RIGHT_BRACKETS] [LEFT_PARENTHESES identifier (COMMA identifier)* RIGHT_PARENTHESES] [for_host] COLON NEWLINE [rule_body]
rule_body: _INDENT (string NEWLINE)* check_expr+ _DEDENT

for_host: FOR identifier

/////////// decorators ////////////
decorators: (AT decorator_expr NEWLINE)+
decorator_expr: identifier [call_suffix]

//////////// type ////////////
type: type_element (OR type_element)*
type_element: schema_type | function_type | basic_type | compound_type | literal_type
schema_type: identifier
function_type: LEFT_PARENTHESES [type_element (COMMA type_element)*] RIGHT_PARENTHESES [RIGHT_ARROW type_element]
basic_type: STRING_TYPE | INT_TYPE | FLOAT_TYPE | BOOL_TYPE | ANY_TYPE
compound_type: list_type | dict_type
list_type: LEFT_BRACKETS (type)? RIGHT_BRACKETS
dict_type: LEFT_BRACE (type)? COLON (type)? RIGHT_BRACE
literal_type: string | number | TRUE | FALSE

//////////// type alias ////////////
type_alias_stmt: TYPE NAME ASSIGN type

//////////// check_stmt ////////////
check_block: CHECK COLON NEWLINE _INDENT check_expr+ _DEDENT
check_expr: simple_expr [IF simple_expr] [COMMA primary_expr] NEWLINE

//////////// mixin_stmt ////////////
mixin_stmt: MIXIN LEFT_BRACKETS [mixins | multiline_mixins] RIGHT_BRACKETS NEWLINE
multiline_mixins: NEWLINE _INDENT mixins NEWLINE _DEDENT
mixins: identifier (COMMA (NEWLINE mixins | identifier))*


//////////// expression_stmt ////////////
expr_stmt: testlist_expr
testlist_expr: test (COMMA test)*
test: if_expr | simple_expr
if_expr: simple_expr IF simple_expr ELSE test

simple_expr: unary_expr | binary_expr | primary_expr

unary_expr: un_op simple_expr
binary_expr: simple_expr bin_op simple_expr

bin_op: L_OR | L_AND
    | OR | XOR | AND
    | SHIFT_LEFT | SHIFT_RIGHT
    | PLUS | MINUS | MULTIPLY | DIVIDE | MOD | DOUBLE_DIVIDE
    | DOUBLE_STAR
    | EQUAL_TO | NOT_EQUAL_TO
    | LESS_THAN | GREATER_THAN | LESS_THAN_OR_EQUAL_TO | GREATER_THAN_OR_EQUAL_TO
    | IN | L_NOT IN | IS | IS L_NOT | L_NOT | AS
un_op: L_NOT | PLUS | MINUS | NOT

primary_expr: identifier call_suffix | operand | primary_expr select_suffix | primary_expr call_suffix | primary_expr slice_suffix
operand: identifier | number | string | constant | quant_expr | list_expr | list_comp | config_expr | dict_comp | schema_expr | lambda_expr | LEFT_PARENTHESES test RIGHT_PARENTHESES
select_suffix: [QUESTION] DOT NAME
call_suffix: LEFT_PARENTHESES [arguments [COMMA]] RIGHT_PARENTHESES
slice_suffix: [QUESTION] LEFT_BRACKETS (test | [test] COLON [test] [COLON [test]]) RIGHT_BRACKETS
arguments: argument (COMMA argument)*
argument: test | NAME ASSIGN test | MULTIPLY test | DOUBLE_STAR test


//////////// operand ////////////
identifier: NAME (DOT NAME)*
quant_expr: quant_op [ identifier COMMA ] identifier IN quant_target LEFT_BRACE (simple_expr [IF simple_expr] | NEWLINE _INDENT simple_expr [IF simple_expr] NEWLINE _DEDENT)? RIGHT_BRACE
quant_target: string | identifier | list_expr | list_comp | config_expr | dict_comp
quant_op: ALL | ANY | FILTER | MAP
list_expr: LEFT_BRACKETS [list_items | NEWLINE [list_items]] RIGHT_BRACKETS
list_items: list_item ((COMMA [NEWLINE] | [NEWLINE]) list_item)* [COMMA] [NEWLINE]
list_item: test | star_expr | if_item
list_comp: LEFT_BRACKETS (list_item comp_clause+ | NEWLINE list_item comp_clause) RIGHT_BRACKETS
dict_comp: LEFT_BRACE (entry comp_clause+ | NEWLINE entry comp_clause+) RIGHT_BRACE
entry: test (COLON | ASSIGN | COMP_PLUS) test
comp_clause: FOR loop_variables [COMMA] IN simple_expr [NEWLINE] [IF test [NEWLINE]]
if_entry: IF test COLON if_entry_exec_block (ELIF test COLON if_entry_exec_block)* (ELSE COLON if_entry_exec_block)?
if_entry_exec_block: (test (COLON | ASSIGN | COMP_PLUS) test | double_star_expr | if_entry) [NEWLINE] | NEWLINE _INDENT (test (COLON | ASSIGN | COMP_PLUS) test | double_star_expr | if_entry) ((COMMA [NEWLINE] | [NEWLINE]) (test (COLON | ASSIGN | COMP_PLUS) test | double_star_expr | if_entry))* [COMMA] [NEWLINE] _DEDENT
if_item: IF test COLON if_item_exec_block (ELIF test COLON if_item_exec_block)* (ELSE COLON if_item_exec_block)?
if_item_exec_block: list_item [NEWLINE] | NEWLINE _INDENT list_item ((COMMA [NEWLINE] | NEWLINE) list_item)* [COMMA] [NEWLINE] _DEDENT

star_expr: MULTIPLY test
double_star_expr: DOUBLE_STAR test
loop_variables: primary_expr (COMMA primary_expr)*
schema_expr: identifier (LEFT_PARENTHESES [arguments] RIGHT_PARENTHESES)? config_expr
config_expr: LEFT_BRACE [config_entries | NEWLINE [config_entries]] RIGHT_BRACE
config_entries: config_entry ((COMMA [NEWLINE] | [NEWLINE]) config_entry)* [COMMA] [NEWLINE]
config_entry: test (COLON | ASSIGN | COMP_PLUS) test | double_star_expr | if_entry

//////////// lambda_expr ////////////
lambda_expr: LAMBDA [schema_arguments] [RIGHT_ARROW type] LEFT_BRACE [expr_stmt | NEWLINE _INDENT schema_init_stmt+ _DEDENT] RIGHT_BRACE

//////////// misc ////////////
number: DEC_NUMBER [multiplier] | HEX_NUMBER | BIN_NUMBER | OCT_NUMBER | FLOAT_NUMBER
multiplier: SI_N_L | SI_U_L | SI_M_L | SI_K_L | SI_K | SI_M | SI_G | SI_T | SI_P
    | SI_K_IEC | SI_M_IEC | SI_G_IEC | SI_T_IEC | SI_P_IEC
string: STRING | LONG_STRING
constant : TRUE | FALSE | NONE | UNDEFINED

// Tokens
ASSIGN: "="
COLON: ":"
SEMI_COLON: ";"
COMMA: ","
QUESTION: "?"
ELLIPSIS: "..."
RIGHT_ARROW: "->"
LEFT_PARENTHESES: "("
RIGHT_PARENTHESES: ")"
LEFT_BRACKETS: "["
RIGHT_BRACKETS: "]"
LEFT_BRACE: "{"
RIGHT_BRACE: "}"
PLUS: "+"
MINUS: "-"
MULTIPLY: "*"
DIVIDE: "/"
MOD: "%"
DOT: "."
AND: "&"
OR: "|"
XOR: "^"
NOT: "~"
LESS_THAN: "<"
GREATER_THAN: ">"
EQUAL_TO: "=="
NOT_EQUAL_TO: "!="
GREATER_THAN_OR_EQUAL_TO: ">="
LESS_THAN_OR_EQUAL_TO: "<="
DOUBLE_STAR: "**"
DOUBLE_DIVIDE: "//"
SHIFT_LEFT: "<<"
SHIFT_RIGHT: ">>"
AT: "@"

COMP_PLUS: "+="
COMP_MINUS: "-="
COMP_MULTIPLY: "*="
COMP_DIVIDE: "/="
COMP_MOD: "%="
COMP_AND: "&="
COMP_OR: "|="
COMP_XOR: "^="
COMP_DOUBLE_STAR: "**="
COMP_DOUBLE_DIVIDE: "//="
COMP_SHIFT_LEFT: "<<="
COMP_SHIFT_RIGHT: ">>="

// Special tokens
IMPORT: "import"
AS: "as"
RULE: "rule"
SCHEMA: "schema"
MIXIN: "mixin"
PROTOCOL: "protocol"
CHECK: "check"
FOR: "for"
ASSERT: "assert"
IF: "if"
ELIF: "elif"
ELSE: "else"
L_OR: "or"
L_AND: "and"
L_NOT: "not"
IN: "in"
IS: "is"
LAMBDA: "lambda"
ALL: "all"
ANY: "any"
FILTER: "filter"
MAP: "map"
TYPE: "type"

ANY_TYPE: "any"
STRING_TYPE: "str"
INT_TYPE: "int"
FLOAT_TYPE: "float"
BOOL_TYPE: "bool"

// Constant tokens
TRUE: "True"
FALSE: "False"
NONE: "None"
UNDEFINED: "Undefined"

// Binary prefix
SI_N_L: "n"
SI_U_L: "u"
SI_M_L: "m"
SI_K_L: "k"
SI_K: "K"
SI_M: "M"
SI_G: "G"
SI_T: "T"
SI_P: "P"
SI_K_IEC: "Ki"
SI_M_IEC: "Mi"
SI_G_IEC: "Gi"
SI_T_IEC: "Ti"
SI_P_IEC: "Pi"
IEC: "i"

NAME: /\$?[a-zA-Z_]\w*/
COMMENT: /#[^\n]*/
NEWLINE: ( /\r?\n[\t ]*/ | COMMENT )+

STRING: /r?("(?!"").*?(?<!\\)(\\\\)*?"|'(?!'').*?(?<!\\)(\\\\)*?')/i
LONG_STRING: /r?(""".*?(?<!\\)(\\\\)*?"""|'''.*?(?<!\\)(\\\\)*?''')/is

DEC_NUMBER: /\-?0|\-?[1-9]\d*/i
HEX_NUMBER: /\-?0[xX][0-9a-fA-F]+/i
OCT_NUMBER: /\-?0[oO]?[0-7]+/i
BIN_NUMBER: /\-?0[bB][0-1]+/i
FLOAT_NUMBER: /(([-+]?\d+\.\d*|\.\d+)(e[-+]?\d+)?|\d+(e[-+]?\d+))/i

%ignore /[\t \f]+/  // WS
%ignore /\\[\t \f]*\r?\n/   // LINE_CONT
%declare _INDENT _DEDENT
```
