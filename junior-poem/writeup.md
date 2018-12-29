# Poet

The CTF is presented as

```
We are looking for the poet of the year:

nc 35.207.132.47 22223

Difficulty estimate: very easy
```

Linked with and executable binary.

When connecting over netcat we are lead through a simple program:

```
**********************************************************
* We are searching for the poet of the year 2018.        *
* Submit your one line poem now to win an amazing prize! *
**********************************************************

Enter the poem here:
> 
```
one can then enter a string, it continues
```
Who is the author of this poem?
> 
```
one can then enter some other string and it goes on
```
+---------------------------------------------------------------------------+
THE POEM
eat sleep pwn
SCORED 300 POINTS.

SORRY, THIS POEM IS JUST NOT GOOD ENOUGH.
YOU MUST SCORE EXACTLY 1000000 POINTS.
TRY AGAIN!
+---------------------------------------------------------------------------+

```

So, we want to get a million points. Running the binary gets the same on a
local command line.

# first stab

`strings poet` shows us some stuff:

```
./flag.txt
ESPR
sleep
repeat
capture
flag
Enter the poem here:
CONGRATULATIONS
THE POET
%.64s
RECEIVES THE AWARD FOR POET OF THE YEAR 2018!
THE PRIZE IS THE FOLLOWING FLAG:
+---------------------------------------------------------------------------+
+---------------------------------------------------------------------------+
THE POEM
%.1024s
SCORED %d POINTS.
Who is the author of this poem?
**********************************************************
* We are searching for the poet of the year 2018.        *
* Submit your one line poem now to win an amazing prize! *
**********************************************************
SORRY, THIS POEM IS JUST NOT GOOD ENOUGH.
YOU MUST SCORE EXACTLY 1000000 POINTS.
TRY AGAIN!
+---------------------------------------------------------------------------+
```

We can conclude: the poet's name should be maximal 64 characters long, the poem 1024.

There are some hard coded words and with some playing we can find out, space
separated words give points if they are in a list of good words:

ESPR, flag, sleep, repeat, capture, flag

From the organisers' slogan, I also tried `eat` and despite not in strings, it
also gives points. Each of these gives 100 points per occurance.

Sadly, counting the trailing space, the shortest is `eat` with 4 characters,
which fits 256 times into 1024 characters (one can try typing more `eat` into
the poem, but really only the first 1024 chars get scored). So the maximum with
simple words is 25600 points, only 974400 points short of the target.

Brute forcing I didn't find any other mechanism in the poem to manipulate the
score (like math operations, control characters, math characters). So I figured
it might be worth reading the disassembly of the evaluation function to look at
its control flow if there is a way to trick the score computation. (But I
didn't reach that point).

# disassembly

`objdump -D poet` reveals a few things

## how to open the flag

`fopen` is used exactly once in the disassembly:

```
0000000000400767 <reward>:
...
  40077a:       e8 d1 fe ff ff          callq  400650 <fopen@plt>
...
```

The rest of the `reward` function looked boring to me, it was just clear that I
need to reach the point where `reward` gets called.

## where's reward called

```
000000000040098b <main>:
  40098b:       53                      push   %rbx
  40098c:       b9 00 00 00 00          mov    $0x0,%ecx
  400991:       ba 02 00 00 00          mov    $0x2,%edx
  400996:       be 00 00 00 00          mov    $0x0,%esi
  40099b:       48 8b 3d de 16 20 00    mov    0x2016de(%rip),%rdi        # 602080 <stdout@@GLIBC_2.2.5>
  4009a2:       e8 99 fc ff ff          callq  400640 <setvbuf@plt>
  4009a7:       48 8d 3d 92 02 00 00    lea    0x292(%rip),%rdi        # 400c40 <_IO_stdin_used+0x1c0>
  4009ae:       e8 4d fc ff ff          callq  400600 <puts@plt>
  4009b3:       48 8d 1d e6 16 20 00    lea    0x2016e6(%rip),%rbx        # 6020a0 <poem>
  4009ba:       b8 00 00 00 00          mov    $0x0,%eax
  4009bf:       e8 71 ff ff ff          callq  400935 <get_poem>
  4009c4:       b8 00 00 00 00          mov    $0x0,%eax
  4009c9:       e8 97 ff ff ff          callq  400965 <get_author>
  4009ce:       b8 00 00 00 00          mov    $0x0,%eax
  4009d3:       e8 df fd ff ff          callq  4007b7 <rate_poem>
  4009d8:       81 bb 40 04 00 00 40    cmpl   $0xf4240,0x440(%rbx)
  4009df:       42 0f 00
  4009e2:       74 0e                   je     4009f2 <main+0x67>
  4009e4:       48 8d 3d 45 03 00 00    lea    0x345(%rip),%rdi        # 400d30 <_IO_stdin_used+0x2b0>
  4009eb:       e8 10 fc ff ff          callq  400600 <puts@plt>
  4009f0:       eb c8                   jmp    4009ba <main+0x2f>
  4009f2:       b8 00 00 00 00          mov    $0x0,%eax
  4009f7:       e8 6b fd ff ff          callq  400767 <reward>
  4009fc:       0f 1f 40 00             nopl   0x0(%rax)
```

This is actually rather readable, the main program calls `get_poem`, then
`get_author`, then `rate_poem` and then eventually calls `reward`.

The function gets called when (to be honest, I'm working with guesses here, in
the main function, I only see one conditional jump (`je` = jump if equal) and
the preceeding comparison compares the hard coded value `0xf4240` (1000000) to
whatever is stored in `0x440`.

Since 1000000 is what the interface claims the desired score should be, we can
assume `0x440` is where the score is stored (and even if not, we are interested
in where `0x440` gets manipulated).

## how is the score computed

Soooo, I started grepping the disassembly for `0x440`. There are two occurances in `rate_poem`:

 - It gets moved somewhere (not manipulated, so don't care) here:

```
  400904:       e9 03 ff ff ff          jmpq   40080c <rate_poem+0x55>
  400909:       8b 15 d1 1b 20 00       mov    0x201bd1(%rip),%edx        # 6024e0 <poem+0x440>
  40090f:       48 8d 35 8a 17 20 00    lea    0x20178a(%rip),%rsi        # 6020a0 <poem>
  400916:       48 8d 3d 83 02 00 00    lea    0x283(%rip),%rdi        # 400ba0 <_IO_stdin_used+0x120>
  40091d:       b8 00 00 00 00          mov    $0x0,%eax
  400922:       e8 e9 fc ff ff          callq  400610 <printf@plt>
```

 - It gets incremented by `0x64` (100) here:
```
  40080a:       eb 21                   jmp    40082d <rate_poem+0x76>
  40080c:       83 05 cd 1c 20 00 64    addl   $0x64,0x201ccd(%rip)        # 6024e0 <poem+0x440>
  400813:       48 8d 35 77 02 00 00    lea    0x277(%rip),%rsi        # 400a91 <_IO_stdin_used+0x11>
  40081a:       bf 00 00 00 00          mov    $0x0,%edi
  40081f:       e8 3c fe ff ff          callq  400660 <strtok@plt>
  400824:       48 85 c0                test   %rax,%rax
  400827:       0f 84 dc 00 00 00       je     400909 <rate_poem+0x152>
```

And there's one in `get_poem`:

```
0000000000400935 <get_poem>:
  400935:       48 83 ec 08             sub    $0x8,%rsp
  400939:       48 8d 3d 7b 01 00 00    lea    0x17b(%rip),%rdi        # 400abb <_IO_stdin_used+0x3b>
  400940:       b8 00 00 00 00          mov    $0x0,%eax
  400945:       e8 c6 fc ff ff          callq  400610 <printf@plt>
  40094a:       48 8d 3d 4f 17 20 00    lea    0x20174f(%rip),%rdi        # 6020a0 <poem>
  400951:       e8 da fc ff ff          callq  400630 <gets@plt>
  400956:       c7 05 80 1b 20 00 00    movl   $0x0,0x201b80(%rip)        # 6024e0 <poem+0x440>
  40095d:       00 00 00
  400960:       48 83 c4 08             add    $0x8,%rsp
  400964:       c3                      retq

```

I found this a bit surprising: Why set the score to zero in `get_poem` and not
at the start of the score computation? What happens after the score gets set to
zero and before the computation and check runs (simply from what I see as
user)? **The poet's name is entered!**

## The poet's name

So let's check where the poet's name is entered:

```
0000000000400965 <get_author>:
  400965:       48 83 ec 08             sub    $0x8,%rsp
  400969:       48 8d 3d a8 02 00 00    lea    0x2a8(%rip),%rdi        # 400c18 <_IO_stdin_used+0x198>
  400970:       b8 00 00 00 00          mov    $0x0,%eax
  400975:       e8 96 fc ff ff          callq  400610 <printf@plt>
  40097a:       48 8d 3d 1f 1b 20 00    lea    0x201b1f(%rip),%rdi        # 6024a0 <poem+0x400>
  400981:       e8 aa fc ff ff          callq  400630 <gets@plt>
  400986:       48 83 c4 08             add    $0x8,%rsp
  40098a:       c3                      retq
```

(Thank you `objdump`) There is one variable accessed, namely `poem+0x400`. So
the poet is just before the score in memory. 64 characters before. Back to
running the application and enter a poet's name that's longer. (NB: iirc
there's somewhere in the code a check if the poem is empty-string, so better
enter something as poem).

Indeed, with a 65-character poet name we get some small score.

## Solving the CTF

Now I *could* convert 1000000 to hex (already in the disassembly) and then pipe
the bits from some sane programming language into netcat … or do some math and
look up in the ascii man page what characters to type … but it was late and I
wasn't super concentrated. So I guesstimated that three characters are about
right for a million, and did binary search on the most significant character,
then on the middle character and then on the least significant character.

The solving poet's name is then `................................................................@B^O`
(that is, `CTRL+O` in the end)

```sh
pseyfert@robusta:/tmp > nc 35.207.132.47 22223

**********************************************************
* We are searching for the poet of the year 2018.        *
* Submit your one line poem now to win an amazing prize! *
**********************************************************

Enter the poem here:
> df
Who is the author of this poem?
> ................................................................@B^O

+---------------------------------------------------------------------------+
THE POEM
df
SCORED 1000000 POINTS.

CONGRATULATIONS

THE POET
................................................................

RECEIVES THE AWARD FOR POET OF THE YEAR 2018!

THE PRIZE IS THE FOLLOWING FLAG:
35C3_f08b903f48608a14cbfbf73c08d7bdd731a87d39

+---------------------------------------------------------------------------+

```


[![Creative Commons License](https://i.creativecommons.org/l/by-sa/4.0/80x15.png) This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-sa/4.0/">Creative Commons Attribution-ShareAlike 4.0 International License](http://creativecommons.org/licenses/by-sa/4.0/)]
