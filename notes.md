FASM:
- https://flatassembler.net/docs.php?article=fasmg (Introduction)
- https://flatassembler.net/docs.php?article=fasmg_manual (Manual)
- https://flatassembler.net/docs.php?article=manual (Other manual)

JONESFORTH:
- https://github.com/nornagon/jonesforth/blob/master/jonesforth.S

# Notes on implementation

This is my summary of the most important parts of
https://raw.githubusercontent.com/nornagon/jonesforth/master/jonesforth.S.

## Dictionary

In Forth, words are stored in a dictionary. The dictionary is a linked list whose entries look like this:

    +------------------------+--------+---------- - - - - +----------- - - - -
    | LINK POINTER           | LENGTH/| NAME              | DEFINITION
    |                        | FLAGS  |                   |
    +--- (4 bytes) ----------+- byte -+- n bytes  - - - - +----------- - - - -

For example, DOUBLE and QUADRUPLE may be stored like this:

      pointer to previous word
       ^
       |
    +--|------+---+---+---+---+---+---+---+---+------------- - - - -
    | LINK    | 6 | D | O | U | B | L | E | 0 | (definition ...)
    +---------+---+---+---+---+---+---+---+---+------------- - - - -
       ^       len                         padding
       |
    +--|------+---+---+---+---+---+---+---+---+---+---+---+---+------------- - - - -
    | LINK    | 9 | Q | U | A | D | R | U | P | L | E | 0 | 0 | (definition ...)
    +---------+---+---+---+---+---+---+---+---+---+---+---+---+------------- - - - -
       ^       len                                     padding
       |
       |
    LATEST

The Forth variable LATEST contains a pointer to the most recently defined word.

## Threaded code

In a typical Forth interpreter, code is stored in a peculiar way. (This way of
storing code is primarily motivated by space contraints on early systems.)

The definition of a word is stored as a sequence of memory adresses of each of
the words making up that definition. (At the end of a compiled definition, there
is also some extra code that causes execution to continue in the correct way.)

We use a register (ESI) to store a reference to the next index of the
word (inside a definition) that we are executing. Then, in order to execute a
word, we just jump to whatever address is pointed to by ESI. The code for
updating ESI and continuing execution is stored at the end of each subroutine.

Of course, this approach only works if each of the words that we are executing
is defined in assembly, but we also want to be able to execute Forth words!

We get around this problem by adding a "codeword" to the beginning of any
compiled subroutine. This codeword is a pointer to the intrepreter to run the
given function. In order to run such functions, we actually need two jumps when
executing: In order to execute a word, we jump to the address at the location
pointed to by the address in ESI.

## Definitions

What does the codeword of a Forth word contain? It needs to save the old value
of ESI (so that we can resume execution of whatever outer definition we are
executing at the time) and set the new version of ESI to point to the first word
in the inner definition.

The stack where the values of ESI are stored is called the "return stack". We
will use EBP for the return stack.

As mentioned, whenever we finish executing a Forth word, we will need to
continue execution in the manner described in the previous section. When the
word being executed is itself written in Forth, we need to pop the old value of
ESI that we saved at the beginning of the definition before doing this.

Thus, the actual data for a word in a dictionary will look something like this:

      pointer to previous word
       ^
       |
    +--|------+---+---+---+---+---+---+---+---+------------+------------+------------+------------+
    | LINK    | 6 | D | O | U | B | L | E | 0 | DOCOL      | DUP        | +          | EXIT       |
    +---------+---+---+---+---+---+---+---+---+------------+--|---------+------------+------------+
       ^       len                         pad  codeword      |
       |                                                      V
      LINK in next word                            points to codeword of DUP

Here, DOCOL (the codeword) is address of the simple interpreter described above,
while EXIT a word (implemented in assembly) that takes care of popping ESI and
continuing execution. Note that DOCOL, DUP, + and EXIT are all stored as
addresses which point to codewords.

## Literals

Literals are handled in a special way. There is a word in Forth, called LIT,
implemented in assembly. When executed, this word looks at the next Forth
instruction (i.e. the value of ESI), and places that on the stack as a literal,
and then manipulates ESI to skip over the literal value.