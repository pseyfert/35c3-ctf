


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
