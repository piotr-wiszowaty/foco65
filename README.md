foco65
======

Forth cross-compiler targeting 6502 processors. It needs a 6502
cross-assembler as a backend.

foco65 should be assembler-neutral, but ...

Usage
-----

foco65 [OPTIONS] INPUT-FILE

OPTIONS:

-h                           display help
-p ADDR,--pstack-bottom=ADDR parameter stack bottom address
-s SECTS,--sections=SECTS    specify comma separated list of sections
                             default: init,boot,data,code
-S INT,--pstack-SIZE=INT     parameter stack size

Words
-----

: @ 0= 1- 1+ 2/ 2* 2@ 2! and c! c@ cmove count d- d+ d= do drop dup i j loop
+loop lshift <= < >= > - + or over rshift rsp sp swap unloop u< u> while <>
= ! [ ] [code] [end-code] cell cells not [section] variable 2variable
constant create , c, ,' ' ," " allot lit \ ( recursive [label] * /

Internals
---------

Type: indirect-threaded.

Cell size is 16 bits.

Parameter stack pointer is kept in register `X`.
Parameter stack pointer is decremented before write (stack push) and
incremented after read (stack pop).

Hardware stack (page $01) is used as return stack.

Instruction pointer and other work registers are kept in
page zero occupying total of 12 bytes.

After initializing interpreter internal state user-defined word
`main` is executed.

Generated code is placed into sections which are output in the
order: `init`, `boot`, `data`, `code` or other - specified by the user.

Example
-------

Atari XL/XE example: display character table as 16x16 array.
Uses xasm as a backend.

<pre><code>
[section] init

[code]
 org $3000
[end-code]

[section] code

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

: ++          ( addr -- )
  [label] plus_plus
  dup @ 1+ swap ! ;
...
  array ++
  array 1 cell + ++
...
</code></pre>
