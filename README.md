foco65
======

Forth cross-compiler targeting 6502 processors. Outputs [xasm](https://github.com/pfusik/xasm)
compatible assembly source code containing Forth runtime and compiled user
program.

Runtime
-------

Generated Forth runtime is 16-bit indirect-threaded.

Parameter stack is full-descending and addressed by `X` register. Address and
size of the parameter stack are by default $600 and 256 respectively and are
user-configurable.

Hardware stack is used as return stack.

The runtime uses 12 bytes on zero page: 2-byte instruction pointer `ip` and
5 2-byte work registers: `w`, `z`, `cntr`, `tmp`, `tmp2` which may be used
in user-defined words (note than on every word entry these registers' contents
are undefined).

On entry runtime initializes instruction pointer and enters user-defined word
`main` which is supposed to never exit (i.e. contain an infinite loop).

Generated Source Code
---------------------

Generated assembly source code for words/data/code blocks is logically grouped
into _sections_. Default order of sections is: `init`, `boot`, `data`, `text`.

Sections:
* `init` - top of the output, typically contains user-provided assembly code
  block with `org` directive
* `boot` - forth runtime
* `data` - user data
* `text` - core words and user program

User-defined data is output into the most recently declared data section (with
`[data-section]` word), everything else is output into the most recently
declared text section (with `[text-section]` word).

Syntax
------

### Identifiers

Identifiers are case sensitive.

Constant and data identifiers consist of alphanumeric characters, '\_'s, '-'s
and '?'s and must start with a letter, '\_', '-' or '?'. In generated assembly
labels every '-' is replaced by '\_' and every '?' is replaced by '\_is\_'.

Word identifiers consist of printable ASCII characters. Word identifiers
consisting only of alphanumeric characters, '\_'s, '-'s, '?'s and starting with
a letter, '\_', '-' or a '?' are translated automatically to valid assembly
labels. Other word identifiers require user-defined label specification (see
below).

Section identifiers consist of printable ASCII characters.

### Comments

Syntax:
<pre><code>
\ this is a one-line comment
( this is a multi-line
  comment)
</code></pre>

### Numbers

Syntax:
<pre><code>
123   \ decimal number
$BA98 \ hexadecimal number
</code></pre>

### Word definitions

Syntax:

`:` _name_ *[* `[label]` _asm-label_ *]* _words_ `;`

or:

`:` _name_
*[* `[label]` _asm-label_ *]*
`[code]`

; inline-assembly

`[end-code]` `;`

Examples:
<pre><code>
: foo begin end-flag until ;
</code></pre>
<pre><code>
: bar
[code]
 lda #1
 sta $D5E8
 jmp next
[end-code] ;
</code></pre>
<pre><code>
: +7 ( n1 -- n2 )
  [label] plus_7
  7 + ;
</code></pre>

### Data declarations

One-cell variable:

`variable` _name_

Two-cell variable:

`2variable` _name_

Assembly label at current program counter:

`create` _name_

Compile a one-cell value:

_value_ `,`

Compile a one-byte value:

_value_ `c,`

Allocate a byte-array:

_length_ `allot`

Allocate Atari XL/XE Antic counted string:

`,'` _text_`'`

Allocate ASCII counted string:

`,"` _text_`"`

Allocate Atari XL/XE Antic string:

`'` _text_`'`

Allocate ASCII string:

`"` _text_`"`

### Constants

Syntax:

_value_ `constant` _name_

Example:
<pre><code>
$230 constant dladr
</code></pre>

### Inline assembly blocks

Syntax:

`[code]`

_; assembly code_

`[end-code]`

### Compiler directives

Syntax:

`[text-section]` _name_

or:

`[data-section]` _name_

Words
-----

`:` `@` `0=` `1-` `1+` `2/` `2*` `2@` `2!` `and` `c!` `c@` `cmove` `count` `d-` `d+` `d=` `do` `drop` `dup` `fill` `i` `j` `loop`
`+loop` `lshift` `<=` `<` `>=` `>` `-` `+` `or` `over` `rshift` `rsp` `sp` `swap` `unloop` `u<` `u>` `while` `<>` `>r` `<r`
`=` `!` `[` `]` `[code]` `[end-code]` `cell` `cells` `not` `[text-section]` `[data-section]`
`variable` `2variable` `constant` `create` `,` `c,` `,'` `'` `,"` `"` `allot`
`lit` `\` `(` `recursive` `[label]` `*` `/` `m*`

Usage
-----

```sh
foco65 [OPTIONS] INPUT-FILE
```

OPTIONS:

    -h                           display help
    -p ADDR,--pstack-bottom=ADDR parameter stack bottom address
    -s SECTS,--sections=SECTS    specify comma separated list of sections
                                 default: init,boot,data,text
    -S INT,--pstack-SIZE=INT     parameter stack size

Example:

```sh
$ foco65 foo.forth > foo.asx
```

Examples
--------

Typical Atari XL/XE executable program structure.

<pre><code>
[text-section] init

[code]
 org $2000
[end-code]

\ constant definitions
\ data declarations
\ word definitions

[text-section] text

: main
  \ user program initialization
  begin
    \ user program main loop
  again ;

[code]
 run boot
[end-code]
</code></pre>

Atari XL/XE example: display character table.

<pre><code>
[text-section] init

[code]
 org $3000
[end-code]

[text-section] text

$230 constant dladr

variable screen
variable cursor
variable line

: cursor-next   ( -- u )
  cursor @ dup 1+ cursor ! ;

: put-char      ( c -- )
  cursor-next c! ;

: set-cursor    ( u -- )
  screen @ + cursor ! ;
  
: main
  dladr @ 4 + @ screen !

  0 line !
  16 0 do
    line @ set-cursor
    line @ 40 + line !
    16 0 do
      i j 4 lshift or put-char
    loop
  loop

  begin again ;

[code]
 run boot
[end-code]
</code></pre>

Atari XL/XE example of defining word in assembly:
wait for keypress and push the pressed key's code
on the parameter stack.

<pre><code>
: get-char    ( -- c )
[code]
 lda #0
 dex
 sta pstack,x
 stx w
 jsr do_gc
 ldx w
 dex
 sta pstack,x
 jmp next

do_gc
 lda $E425
 pha
 lda $E424
 pha
 rts
[end-code] ;
</code></pre>

Increase cell at the given address. Shows defining words not
being a valid assembler label.

<pre><code>
create array 16 cells allot

\ increase cell at given address
: ++          ( addr -- )
  [label] plus_plus
  dup @ 1+ swap ! ;
</code></pre>
