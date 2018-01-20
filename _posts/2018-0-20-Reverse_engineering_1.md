---
layout: post
title: RE-001 Introduction au reverse engineering avec Radare2
---

![radare2-1000x450.jpg]({{ site.baseurl }}/images/radare2-1000x450.jpg{:class="img-responsive"}


# Reverse enginering 

Dans ce billet nous allons plonger dans le monde fascinant de l'ingénierie inverse en cassant puis en patchant un binaire simple.

Nous utiliserons également Radare2 qui est le plus nerd des outil de RE actuel (de plus il est open-source).

Pour ce qui est du binaire, il s'agit du challenge 2013-BIN-50 du site http://backdoor.sdslabs.co/.

> Ici on travailleras avec un binaire ELF 64 donc, sous linux.

## Une technique basique 

Pour une première approche on va récupérer des informations sur notre exécutable avec Rabin2 : 

```bash
rabin2 -I binaary50                                                       arch     x86
binsz    13381
bintype  elf
bits     64
canary   true
class    ELF64
crypto   false
endian   little
havecode true
intrp    /lib64/ld-linux-x86-64.so.2
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
```

On voit ici que le binaire est un ELF64 codé en C avec une stack non exécutable. Il va donc ici être plutôt compliqué de faire une injection par buffer-overflow.

Analysons les fonctions importés : 

```bash
rabin2 -iz binaary50 
[Imports]
   1 0x00400000    WEAK  NOTYPE __gmon_start__
   2 0x00400000    WEAK  NOTYPE _Jv_RegisterClasses
   3 0x004006f0  GLOBAL    FUNC sym.imp.std::ios_base::Init::Init()
   4 0x00400700  GLOBAL    FUNC __libc_start_main
   5 0x00400710  GLOBAL    FUNC __cxa_atexit
   6 0x00400730  GLOBAL    FUNC sym.imp.std::basic_ostream<char,std::char_traits<char>>&std::operator<<<std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>>&,charconst*)
   7 0x00400740  GLOBAL    FUNC strlen
   8 0x00400750  GLOBAL    FUNC __stack_chk_fail
   9 0x00400760  GLOBAL    FUNC sym.imp.std::ostream::operator<<(std::ostream&(*)(std::ostream&))
  10 0x00400770  GLOBAL    FUNC sym.imp.std::basic_ostream<char,std::char_traits<char>>&std::endl<char,std::char_traits<char>>(std::basic_ostream<char,std::char_traits<char>>&)
  11 0x00400720  GLOBAL    FUNC sym.imp.std::ios_base::Init::~Init()
   1 0x00400000    WEAK  NOTYPE __gmon_start__
   2 0x00400000    WEAK  NOTYPE _Jv_RegisterClasses
000 0x00000cd0 0x00400cd0  25  26 (.rodata) ascii Password is Advicemallard
001 0x00000cf0 0x00400cf0  36  37 (.rodata) ascii qie////3213/wqeqwe/qwqweqsxcf/d/////
002 0x00000d15 0x00400d15  18  19 (.rodata) ascii Password is Butter
003 0x00000d28 0x00400d28  22  23 (.rodata) ascii Password is Hoobastank
004 0x00000d3f 0x00400d3f  17  18 (.rodata) ascii Password is Darth
005 0x00000d51 0x00400d51  22  23 (.rodata) ascii Password is Jedimaster
006 0x00000d68 0x00400d68  23  24 (.rodata) ascii Password is Masternamer
007 0x00000d80 0x00400d80  20  21 (.rodata) ascii Password is Morpheus
008 0x00000d95 0x00400d95  19  20 (.rodata) ascii Password is Neutron
009 0x00000da9 0x00400da9  18  19 (.rodata) ascii Password is Coyote
010 0x00000dbc 0x00400dbc  18  19 (.rodata) ascii Password is Tweety
011 0x00000dcf 0x00400dcf  20  21 (.rodata) ascii Nothing to see here.
012 0x00000de4 0x00400de4  28  29 (.rodata) ascii Please provide the password 
```

Un détail intéressant met en avant plusieurs chaînes de caractères parlant d'un mot de passe.
Étant feignant on va ici écrire un script qui test chacune des chaînes trouvé avec rabin2 : 

```python
#!/usr/bin/python3

import os
for i in open("dict.txt","r"):
	out=os.popen("./binaary50 {}".format(i))
    out=out.read()
    if out not in "Nothing to see here\n":
    	print("The flag is {}".format(out)) 
```

On lance notre script magique : 

```bash
./pwn.py                     
The flag is 3cd50c6be9bbede06e51741928d88b7e
```

Bingo ! On va maintenant faire un sha256 de ce flag pour le soumettre au site :

```bash
echo -n "3cd50c6be9bbede06e51741928d88b7e" | sha256sum
dad827e94c609b76424287f2523b2117475df29e4ca8475444a9976faedc00f7
```

> Ici on ajoute `-n` pour supprimer le retour à la ligne que `echo` génère par génère.

Mais cette technique est trop simple !



## A la recherche de STRCMP 

Démarrons radare2 et  analysons le binaire :

```bash
radare2 -w binaary50                                   
 -- For a full list of commands see `strings /dev/urandom`
[0x00400780]> aaaa
[x] Analyze all flags starting with sym. and entry0 (aa)
[x] Analyze len bytes of instructions for references (aar)
[x] Analyze function calls (aac)
[x] Emulate code to find computed references (aae)
[x] Analyze consecutive function (aat)
[x] Constructing a function name for fcn.* and sym.func.* functions (aan)
[x] Type matching analysis for all functions (afta)
```
Cherchons STRCMP dans le main :


```bash
|     ``--> 0x00400ad3      c645e04d       mov byte [local_20h], 0x4d  ; 'M' ; 77
|       |   0x00400ad7      c645e161       mov byte [local_1fh], 0x61  ; 'a' ; 97
|       |   0x00400adb      c645e273       mov byte [local_1eh], 0x73  ; 's' ; 115
|       |   0x00400adf      c645e374       mov byte [local_1dh], 0x74  ; 't' ; 116
|       |   0x00400ae3      c645e465       mov byte [local_1ch], 0x65  ; 'e' ; 101
|       |   0x00400ae7      c645e572       mov byte [local_1bh], 0x72  ; 'r' ; 114
|       |   0x00400aeb      c645e66e       mov byte [local_1ah], 0x6e  ; 'n' ; 110
|       |   0x00400aef      c645e761       mov byte [local_19h], 0x61  ; 'a' ; 97
|       |   0x00400af3      c645e86d       mov byte [local_18h], 0x6d  ; 'm' ; 109
|       |   0x00400af7      c645e965       mov byte [local_17h], 0x65  ; 'e' ; 101
|       |   0x00400afb      c645ea72       mov byte [local_16h], 0x72  ; 'r' ; 114
|       |   0x00400aff      488b45d0       mov rax, qword [local_30h]
|       |   0x00400b03      4883c008       add rax, 8
|       |   0x00400b07      488b00         mov rax, qword [rax]
|       |   0x00400b0a      488d55e0       lea rdx, [local_20h]
|       |   0x00400b0e      4889d6         mov rsi, rdx
|       |   0x00400b11      4889c7         mov rdi, rax
|       |   0x00400b14      e8d2fdffff     call sym.strcmpr_char__char
|       |   0x00400b19      85c0           test eax, eax
|       |   0x00400b1b      0f94c0         sete al
|       |   0x00400b1e      84c0           test al, al
|      ,==< 0x00400b20      7407           je 0x400b29
|      ||   0x00400b22      e83dfdffff     call sym.flag
|     ,===< 0x00400b27      eb3a           jmp 0x400b63
|     |||      ; JMP XREF from 0x00400b20 (main)
|     |`--> 0x00400b29      becf0d4000     mov esi, str.Nothing_to_see_here. ; 0x400dcf ; "Nothing to see here."
```

A l'adresse `0x00400b14` on voit notre fameux strcmp et juste au dessus la suite de caractères `Masternamer`.
Testons avec ça :

```bash
./binaary50 Masternamer           
3cd50c6be9bbede06e51741928d88b7e

```
On y est ! Mais la aussi, trop simple !

## Let's patch this binary : 

Sur le précédent dissasembly nous avons vue qu'a `0x00400b14` se trouve un strcmp :

```
|       |   0x00400b14      e8d2fdffff     call sym.strcmpr_char__char
|       |   0x00400b19      85c0           test eax, eax
|       |   0x00400b1b      0f94c0         sete al
|       |   0x00400b1e      84c0           test al, al
|      ,==< 0x00400b20      7407           je 0x400b29
|      ||   0x00400b22      e83dfdffff     call sym.flag
|     ,===< 0x00400b27      eb3a           jmp 0x400b63
|     |||      ; JMP XREF from 0x00400b20 (main)
|     |`--> 0x00400b29      becf0d4000     mov esi, str.Nothing_to_see_here. ; 0x400dcf ; "Nothing to see here."

```

Si on regarde juste en dessous on voit une instruction `JE` qui saute vers une string "No    thing to see here." .
En testant un mot de passe erroné on obtiens ceci :

```bash
./binaary50 Masternam  
Nothing to see here.
```

On apprend donc que cette instruction permet de définir si oui ou non on a rentré le bon mot de passe. 
Si nous le remplacions par autre chose ? Comme un `JMP` vers l'instruction suivante ? 

```bash
╭─bork-fomb@borkfomb-T450 ~/Documents/CTF  
╰─$ radare2 -w binaary50              
 -- Trace register changes while debugging with 'e trace.cmtregs=true'
[0x00400780]> wa jmp 0x400b22 @0x00400b20
Written 2 byte(s) (jmp 0x400b22) = wx eb00
[0x00400780]> exit
```
On teste : 

```bash
╭─bork-fomb@borkfomb-T450 ~/Documents/CTF  
╰─$ ./binaary50 eee                   
3cd50c6be9bbede06e51741928d88b7e
```



## Conclusion 

Nous avons vue ici 3 manières basiques de casser un binaire. 
Évidemment ce binaire est très basique et simple à détourner mais il est un fondement basique à l'utilisation du logiciel Radare2. 
Dans les prochains chapitres nous parlerons de challenges plus complexes et pourquoi pas de ROP. 
