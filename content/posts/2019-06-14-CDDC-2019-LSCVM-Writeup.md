---
title:  "CDDC 2019 LSCVM Writeup"
date:   2019-06-14 00:00:00
author: "David"
tags:
    - CTF
    - Reverse Engineering
---

This is actually a challenge from the qualifiers, but because this challenge will be used in the Finals again so we weren't allowed to post about it until after the finals. More on that [later](#rant), here's the writeup.

## LSCVM: Immaculate Invasion

> DESCRIPTION
>
> During our recon on the notorious fools of LightSpeedCorp, we have discovered this service which runs on a really simple, tiny, trivial, virtual machine that they have created. We are not sure of its purpose, but it certainly looks fun to play with.
>
> **lscvm-ii.cddc19q.ctf.sg 9001**
>
> IMPORTANT: We highly recommend you to fully understand this challenge as we will use this VM again in the Final.

### Solution

Looking at the strings in the binary, it looks like we are dealing with a Stack based vm, and those long strings are probably the instructions.

![Strings](/2019-06-14-CDDC-2019-LSCVM-Writeup/lscvm_strings.png)

If we try running it we'll get an error.

```
cddc/re/LSCVM ➜ ./lscvm-ii
[-] Flag file open error: No such file or directory
```

We find the code that is throwing the error and we can see it is trying to read a file called flag.

![Read flag](/2019-06-14-CDDC-2019-LSCVM-Writeup/read_flag.png)

Let's create a file called flag to fix this error.
```bash
echo 'CTF{flag}' > flag
```

Now when we run it we can see that it asks us for an ID which we have to figure out.
```
cddc/re/LSCVM ➜ ./lscvm-ii

=== Welcome to LSCVM(LightSpeed Corp Virtual Machine) ===

ID : 1
[-] Wrong id
```

After doing some analysis, I figured out that if argc is 2 the program will print out 'debug' information.
```
cddc/re/LSCVM ➜ ./lscvm-ii a
@0 c [ 02 ]
@1 f [ 02 05 ]
@2 M [ 0a ]
@3 c [ 0a 02 ]
@4 f [ 0a 02 05 ]
@5 M [ 0a 0a ]
@6 h [ 0a 0a 07 ]
@7 i [ 0a 0a 07 08 ]
@8 M [ 0a 0a 38 ]
@9 f [ 0a 0a 38 05 ]
@10 A [ 0a 0a 3d ]
.
.
.
```

The information can be interpreted this way.

| Instruction pointer | instruction | output | Stack after running instruction |
|---|---|---|---|
| @0 | c  | | [ 02 ] |
| @1 | f | | [ 02 05 ] |
| @2 | M | | [ 0a ] |

Using the debug output and static analysis, I was able to recover the whole instruction set.

| Opcode (Hex) | Opcode (Char) | Assmebly | Comment |
|---|---|---|---|
| Default | NIL | nop | no operation, waste cycle |
| 0x0a | \n | nop | no operation, waste cycle |
| 0x20 | \s | nop | no operation, waste cycle |
| 0x41 | A | add | Pop 2 values from stack and push the addition |
| 0x42 | B | hlt | Stop executing code |
| 0x43 | C | jmp | Pop and jump to value |
| 0x44 | D | pop | Pop a value and do nothing |
| 0x45 | E | read | Pop addr and push val. `0` <= addr <= `0x3fff` |
| 0x46 | F | sclone | pop value n and (clone) push n+1 th previous stack value |
| 0x47 | G | ipadd | Pop value and add it to IP (Instruction Pointer) |
| 0x48 | H | sshift | pop value n and shift n+1 th previous stack value to the top of the stack |
| 0x49 | I | pint | Pop value and Print as int |
| 0x4a | J | cmp | pop 2 values and push 0 if equal, 1 if first pop is smaller else -1. `0` |
| 0x4b | K | write | first pop is addr, 2nd pop is value to write |
| 0x4d | M | mul | Pop 2 values from stack and push the multiplication |
| 0x50 | P | pchar | Pops value from stack and print as char |
| 0x52 | R | rjmp | Jump to previous jump location, cant do twice in a row, because it consumes the previous jump location. |
| 0x53 | S | sub | subtract 1st pop from 2nd pop push result |
| 0x56 | V | div | Divide (floor) 2nd pop by 1st pop and push result |
| 0x5a | Z | ipcadd | conditional add to IP, if 2nd pop is 0, add 1st pop to IP |
| 0x61 | a | p0 | Push 0x00 to stack |
| 0x62 | b | p1 | Push 0x01 to stack |
| 0x63 | c | p2 | Push 0x02 to stack |
| 0x64 | d | p3 | Push 0x03 to stack |
| 0x65 | e | p4 | Push 0x04 to stack |
| 0x66 | f | p5 | Push 0x05 to stack |
| 0x67 | g | p6 | Push 0x06 to stack |
| 0x68 | h | p7 | Push 0x07 to stack |
| 0x69 | i | p8 | Push 0x08 to stack |
| 0x6a | j | p9 | Push 0x09 to stack |

The input we are giving the program is actually being executed by the vm as code.

To get the flag, we have to provide the vm with code that will write `lsc_user` to the vm memory, and then provide another code that will write `hi_darkspeed-corp!` to the vm memory.

![Get flag conditions](/2019-06-14-CDDC-2019-LSCVM-Writeup/get_flag.png)

Using the instruction set recovered I wrote this script to generate the vm code required to write the strings to memory and submit it to the service.
```python
#!/usr/bin/env python3
import socket

nums = {
0x00:'a',
0x01:'b',
0x02:'c',
0x03:'d',
0x04:'e',
0x05:'f',
0x06:'g',
0x07:'h',
0x08:'i',
0x09:'j',
0x0a:'jbA',
0x0b:'jcA',
0x0c:'jdA',
0x0d:'jeA',
0x0e:'jfA',
0x0f:'jgA',
0x10:'jhA',
0x11:'jiA',
0x12:'jjA',
0x21:'eiMbA',
0x2d:'jfM',
0x5f:'jjMchMA',
0x61:'jjMcjMAcS',
0x63:'jjMcjMA',
0x64:'jjMcjMAbA',
0x65:'jjMcjMAcA',
0x68:'jjMdhMcAA',
0x69:'jjMdhMdAA',
0x6b:'jjMdjMAbS',
0x6c:'jjMdjMA',
0x6f:'jjMdjMAdA',
0x70:'jjMfhMeSA',
0x72:'jjMfhMcSA',
0x73:'jjMfhMbSA',
0x75:'jjMfhMbAA'}

def write_str(target):
    output = ''
    for i in range(len(target)):
        output += nums[ord(target[i])] + nums[i]

    return output + 'K' * len(target)

id_code = write_str('lsc_user')
pass_code = write_str('hi_darkspeed-corp!')
print('ID: {}\nPassword: {}\n'.format(id_code, pass_code))

print('Connecting to service...')
client=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
client.connect(('lscvm-ii.cddc19q.ctf.sg',9001))
client.sendall('{}\n'.format(id_code).encode())
client.sendall('{}\n'.format(pass_code).encode())
while True:
    msg = client.recv(1024).decode()
    if msg:
        print(msg)
        if '$CDDC19${' in msg:
            client.close()
            break
```

```
cddc/re/LSCVM ➜ ./solve.py
ID: jjMdjMAajjMfhMbSAbjjMcjMAcjjMchMAdjjMfhMbAAejjMfhMbSAfjjMcjMAcAgjjMfhMcSAhKKKKKKKK
Password: jjMdhMcAAajjMdhMdAAbjjMchMAcjjMcjMAbAdjjMcjMAcSejjMfhMcSAfjjMdjMAbSgjjMfhMbSAhjjMfhMeSA
ijjMcjMAcAjjjMcjMAcAjbAjjMcjMAbAjcAjfMjdAjjMcjMAjeAjjMdjMAdAjfAjjMfhMcSAjgAjjMfhMeSAjhAeiMbAjiAKK
KKKKKKKKKKKKKKKK

Connecting to service...


=== Welcome to LSCVM(LightSpeed Corp Virtual Machine) ===

ID : Password :
Login Successful! $CDDC19${IcY_GrE37ings_Fr0M_LigHT5pEeDC0Rp}

lsc_user, Good Bye!
```

### Flag
```
$CDDC19${IcY_GrE37ings_Fr0M_LigHT5pEeDC0Rp}
```

## Rant

The organizers stated in the qualifiers LSCVM challenges that it is important to fully understand the VM as it will be used again in the Finals.

> IMPORTANT: We highly recommend you to fully understand this challenge as we will use this VM again in the Final.

The way they brought this accross made me think that it will play a big part in the finals, so I built a [tool](https://github.com/PotatoDrug/LSCVM-Tool) for the purpose of working on LSCVM challenge(s) during the Finals. However LSCVM was not a big part of the Finals and I didn't get to use my tool at all.

The finals also had Rings where you have to solve a certain number of challenges to unlock the next ring. Personally I don't think the rings concept is a good idea, because what if it just so happens the participant can't solve the starting challenges but they can solve the ones in the next or next next ring? I would've preferred it if the challenges were all available at the start.

I understand that it was implemented to limit the number of teams that can attempt the hardware challenges because they don't have enought equipment for everyone to attempt at the same time, but this can also be done by only unlocking the hardware challenges once a team reach a certain number of points/solves, instead of implementing Rings.

There's not gonna be any writeup for the finals challenges cause I did everything on the provided laptop and didn't transfer the files out.
