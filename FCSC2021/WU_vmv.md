# **FCSC 2021 - Write-up**

*Par loulous24*

## **vmv**

#### **Décompilation du C**

J'ai hésité à quel challenge de reverse j'allais expliquer. J'ai jeté mon dévolu sur `vmv`. Je suis encore assez jeune en reverse donc je ne sais pas trop par quoi commencer. Le but de ce type de challenge est de comprendre des instructions depuis l'assembleur. Heureusement, depuis les challenges de l'année dernière, j'ai appris l'existence de `Ghidra`. `radare` c'est cool mais `Ghidra` c'est vraiment fantastique. On peut définir des structs soi même et ça refait l'analyse automatiquement.

On remarque assez vite une fonction qui ressemble à un main à l'adresse `0x101d44`. Après avoir lu tout le code, voici ce que j'obtiens. Je vais détailler quelques parties intéressantes.

```c
int main(int argc,char **argv)

{
  int instr;
  char *asm_code;
  size_t len_param;
  char *concat_random_input;
  int *input_asm;
  conteneur *container;
  code *function;
  ulong compteur;
  
  if (argc < 2) {
    fprintf(stderr,"usage: %s input\n", argv);
    exit(1);
  }
  len_param = strlen(argv[1]);
  if (len_param != 0x10) {
    fwrite("input must be of size 16",1,0x18,stderr);
    exit(1);
  }
  chain_ptr = (chained_list **)calloc(8,0x20000);
  assign_func_addr(0xadc52d,SUM);
  assign_func_addr(0x560729d,MUL);
  assign_func_addr(0x48c5ccc6,XOR);
  assign_func_addr(0x542010a0,AND);
  assign_func_addr(0xbdecfe55,RET);
  assign_func_addr(0x41f93b4b,EXIT);
  assign_func_addr(0x5e64bb6c,PUSH_imm);
  assign_func_addr(0xed4e2cfb,JUMP);
  assign_func_addr(0x180bc12d,JEQ);
  assign_func_addr(0x5a0f38fc,JNE);
  assign_func_addr(0x27497906,EXIT);
  assign_func_addr(0xba1116a9,ADD_imm);
  assign_func_addr(0xfa83fa5e,CALL);
  assign_func_addr(0x818cd6b5,END);
  assign_func_addr(0x8d67bae1,INPUT);
  assign_func_addr(0xd1450d67,PRINT);
  assign_func_addr(0x8ea45b38,DEC);
  assign_func_addr(0xf00bb6c1,INC);
  assign_func_addr(0x5991ba22,PUSH);
  assign_func_addr(0x43ae1f53,MOD);
  assign_func_addr(0x8960888a,POP);
  assign_func_addr(0x1f0a8e6f,ADD);
  assign_func_addr(0x466a54d9,EXIT);
  assign_func_addr(0xfb521a9c,WRITE);
  assign_func_addr(0xc650f15d,READ);
  asm_code = (char *)base64(s_lwMAAAI7q4cABAAAm1TXOxaSiX4Axmju_001050a0,0x3f10);
  len_param = strlen(argv[1]);
  concat_random_input = (char *)calloc(1,len_param + 0x2f4c);
  input_asm = (int *)base64(s_qRYRugAEAAAiupFZc9ZraYqIYIlrVBk9_00108fc0,0x1328);
  memcpy(concat_random_input,asm_code,0x2f4c);
  len_param = strlen(argv[1]);
  memcpy(concat_random_input + 0x2f4c,argv[1],len_param);
  container = init_data(input_asm,concat_random_input);
  puts("[fr] Ne quittez pas, un correspondant va prendre votre appel... [\\fr]");
  compteur = 0;
  while (container->continue != false) {
    instr = next_instruction(container);
    function = (code *)find_element_in_chain(instr);
    (*function)(container);
    compteur = compteur + 1;
    if (((compteur & 0xfffffff) == 0) && (container->have_printed != true)) {
      puts("[fr] Ne quittez pas, un correspondant va prendre votre appel... [\\fr]");
    }
  }
  return 0;
}
```
En gros, on a `chain_ptr` qui est une table de hashage (c'est l'équivalent d'un dictionnaire en python) qui associe à ce qui ressemble à un opcode, le code d'une fonction. Dans la dernière boucle, on continue à avancer et à éxecuter chaque opcode. La struct `container` contient un pointeur vers un tas et un ensemble de variable qui simulent les registres d'un processeur. Toutes les instructions sont ajoutés à la liste des instructions possibles au début du code.

J'ai modifié leurs noms pour rendre compte le plus fidèlement possible de leur fonctionnement. Seule l'instruction `MUL` était très étrange, nous y reviendrons plus tard. Il ne s'agit pas réellement d'une multiplication. Certaines instructions prennent leurs arguments depuis la "pile", d'autres vont lire dans un des registres ou dans la "RAM". Le nouveau "code" assembleur est la traduction en base64 d'une `string` commençant par `"lwMAAAI7q4cABAA"`.

L'instruction `INPUT` va chercher dans la `string` encodée en base64 commençant par `"qRYRugAEAAAiupFZc9Z"`.

#### **Désassemblage du pseudo-assembleur**

Il est temps de traduire nos opcodes en code assembleur. J'ai codé un petit outil en python pour ça.

```python
code = "a91611ba0004000022ba..."

opcodes = {
"00adc52d" : ("SUM", 0),
"0560729d" : ("MUL", 0),
"48c5ccc6" : ("XOR", 0),
"542010a0" : ("AND", 0),
"bdecfe55" : ("RET", 0),
"41f93b4b" : ("EXIT", 1),
"5e64bb6c" : ("PUSH", 1),
"ed4e2cfb" : ("JUMP", 3),
"180bc12d" : ("JEQ", 3),
"5a0f38fc" : ("JNE", 3),
"27497906" : ("EXIT", 1),
"ba1116a9" : ("ADD", 1),
"fa83fa5e" : ("CALL", 3),
"818cd6b5" : ("END", 2),
"8d67bae1" : ("INPUT", 2),
"d1450d67" : ("PRINT", 2),
"8ea45b38" : ("DEC", 2),
"f00bb6c1" : ("INC", 2),
"5991ba22" : ("PUSH", 2),
"43ae1f53" : ("MOD", 2),
"8960888a" : ("POP", 2),
"1f0a8e6f" : ("ADD", 2),
"466a54d9" : ("EXIT", 1),
"fb521a9c" : ("WRITE", 2),
"c650f15d" : ("READ", 2)}

def transform(opcode):
    return opcode[6:8] + opcode[4:6] + opcode[2:4] + opcode[0:2]

i = 0
while i < len(code)//8:
    op = transform(code[8*i:8*i+8])
    i += 1
    label, var = opcodes[op]
    if var == 0:
        print(f"{i-1:03}: {label}")
    else:
        v = int(transform(code[8*i:8*i+8]), 16)
        i += 1
        if var == 3:
            if v > 0x7FFFFFFF:
                v -= 0x100000000
            v += i
            print(f"{i-2:03}: {label} {v}")
        elif var == 1:
            print(f"{i-2:03}: {label} {v:08x}")
        else:
            v = (v ^ 0x4a0cbe95) & 0x3f
            print(f"{i-2:03}: {label} var_{v}")
```

Les instructions mis en tuple avec un 0 prennent leurs arguments depuis la pile, celles avec un 1 prennent une valeur directement depuis le code (c'est un équivalent d'une valeur immédiate). Les instructions avec un 2 gèrent les variables des registres et celles avec un 3 correspondent à des adresses d'autres endroits du code (pour les saut et les appels). On obtient ce pseudo-assembleur :

```
000: ADD 00000400
002: PUSH var_38
004: POP var_62
006: PUSH 00000011
008: POP var_38
010: ADD var_38
012: PUSH var_38
014: POP var_45
016: INPUT var_46
018: ADD var_46
020: PUSH var_38
022: POP var_32
024: PUSH var_45
026: PUSH 00000000
028: SUM
029: POP var_38
031: WRITE var_32
033: PUSH 00000000
035: POP var_47
037: PUSH var_46
039: PUSH var_47
041: JEQ 58
043: INPUT var_43
045: PUSH var_32
047: PUSH var_47
049: SUM
050: POP var_38
052: WRITE var_43
054: INC var_47
056: JUMP 37
058: PUSH 08929d1e
060: POP var_44
062: PUSH 39ee8310
064: POP var_37
066: CALL 880
068: PUSH var_46
070: PUSH 952db75f
072: JEQ 814
074: PUSH var_46
076: PUSH 140c2cf8
078: JEQ 755
080: PUSH var_46
082: PUSH 4517cc48
084: JEQ 860
086: PUSH var_46
088: PUSH e80c7be2
090: JEQ 442
092: PUSH var_46
094: PUSH 950885b3
096: JEQ 822
[...]
880: PUSH var_45
882: PUSH 00000000
884: SUM
885: POP var_38
887: PUSH var_38
889: READ var_38
891: PUSH var_38
893: READ var_46
895: PUSH 00000001
897: SUM
898: POP var_43
900: POP var_38
902: WRITE var_43
904: RET
905: PUSH 39ee8310
907: XOR
908: PUSH 0000003f
910: AND
911: PUSH var_45
913: SUM
914: POP var_38
916: READ var_46
918: RET
```

#### **Analyse de l'assembleur**

J'ai coupé tout une partie mais là contrairement à un `ELF` classique, on ne peut plus décompiler ça pour réobtenir du code `C` mais ça se lit plutôt bien. Et là, on comprend qu'on a affaire à un bon challenge... En fait, ce code lit l'entrée et en fonction de l'entrée calcule une fonction. **Il s'agit d'une deuxième machine à pile et registre qui fonctionne avec des opcodes.**

On se demande alors où la folie du concepteur du sujet va nous mener (spoiler : loin) et on continue ^^



Le début du code consiste à parser l'entrée pour connaître le nombre de bits à lire puis consiste à lire l'opcode suivant (à l'aide de la fonction à l'adresse `880`). Ensuite, on rentre dans une des fonctions. Après les avoir toutes lues, on se rend compte qu'elles sont très similaires (en terme d'argument, dans la pile ou la RAM) aux instructions de la surmachine virtuelle en code `C`. Elles font d'ailleurs appel à leurs surfonctions pour faire certains calculs. Ainsi, on comprend que la chaîne en `base64` qui lui est passée en argument est en fait une autre suite d'opcode pour une autre machine assembleur.

On décompile le code assembleur avec le code python précédent et modifié un peu pour rendre compte de certains changement, notamment une instruction `PRINT` qui prend en argument un registre simulé et une autre instruction `PRINT` qui lit une valeur immédiate (mais fait un XOR avec une valeur hardcodée aléatoire).

Et là, **oh stupeur**, on commence à rentrer dans la matrice. Il s'agit encore d'un autre simulateur assembleur qui ressemble au précédent mais qu'il lit l'entrée du reste de la string. C'est un pseud-assembleur qui d'une certaine manière s'execute lui même. Seuls les opcodes sont différents (mais pointent au même endroit), la valeur de XOR aléatoire pour le string et le xor avec les opcodes pour le calcul des registres simulés.

![Here we go again](images/here_we_go_again.png)

Heureusement, la bonne idée c'est de scripter le décodage récursif parce qu'on a encore cette simulation récursive **3 fois** !! On comprend pourquoi le binaire initial est très lent !!

```python
func = {
814: ("END", 2),
755: ("DEC", 2),
860: ("RET", 0),
442: ("READ", 2),
822: ("CALL", 3),
214: ("PRINT", 2),
273: ("JMP", 3),
608: ("XOR", 0),
575: ("MUL", 0),
476: ("AND", 0),
730: ("INC", 2),
299: ("JEQ", 3),
415: ("WRITE", 2),
542: ("SUM", 0),
224: ("PUSH", 1),
252: ("INPUT", 2),
236: ("PUSH", 2),
357: ("JNE", 3),
641: ("PRINT", 1),
657: ("EXIT", 1),
661: ("ADD", 1),
680: ("ADD", 2),
703: ("POP", 2),
780: ("MOD", 2)}

def transform(opcode):
    return opcode[6:8] + opcode[4:6] + opcode[2:4] + opcode[0:2]

xor_variable = 0x39ee8310
for j in range(4):
    i = 0
    code = code[8:]
    jump_array = []

    while i < len(code)//8:
        op = transform(code[8*i:8*i+8])
        i += 1
        if not op in opcodes.keys():
            i -= 1
            break
        label, var = func[opcodes[op]]
        if var == 0:
            print(f"{i-1:03}: {label}")
            continue
        v = int(transform(code[8*i:8*i+8]), 16)
        i += 1
        if var == 3:
            if v > 0x7FFFFFFF:
                v -= 0x100000000
            v += i
            if 69 <= i <= 212:
                jump_array.append(v)
            print(f"{i-2:03}: {label} {v}")
        elif var == 1:
            # if i-2 == 58:
                # print(f"0x{v:08x}") # pour afficher la valeur XORée avec l'input
            if i-2 == 62:
                new_xor_variable = v
            if 68 <= i <= 210:
                jump_array.append(f"{v:08x}")
            print(f"{i-2:03}: {label} {v:08x}")
        else:
            v = (v ^ xor_variable) & 0x3f
            print(f"{i-2:03}: {label} var_{v}")

    code = code[8*i:]
    new_opcodes = {}
    if j == 3:
        break
    for k in range(0, len(jump_array), 2):
        new_opcodes[jump_array[k]] = jump_array[k+1]
    opcodes = new_opcodes
    xor_variable = new_xor_variable
    print('\n\n')
```

On arrive à ce code assembleur différent des précédents

```
000: PUSH 00000000
002: POP var_16
004: INPUT var_14
006: PUSH 00000001
008: PUSH var_14
010: PUSH 117052c0
012: MUL
013: JNE 17
015: INC var_16
017: INPUT var_14
019: PUSH var_14
021: PUSH 000077f3
023: POP var_15
025: MOD var_15
027: PUSH 00004926
029: JEQ 35
031: PUSH 00000000
033: POP var_16
035: PUSH var_14
037: PUSH 00007c49
039: POP var_15
041: MOD var_15
043: PUSH 00003159
045: JEQ 51
047: PUSH 00000000
049: POP var_16
051: INPUT var_14
053: PUSH 00000001
055: PUSH var_14
057: PUSH 278bce9d
059: MUL
060: JNE 64
062: INC var_16
064: INPUT var_14
066: PUSH var_14
068: PUSH 000077f3
070: POP var_15
072: MOD var_15
074: PUSH 000028b2
076: JEQ 82
078: PUSH 00000000
080: POP var_16
082: PUSH var_14
084: PUSH 00007c49
086: POP var_15
088: MOD var_15
090: PUSH 000044a9
092: JEQ 98
094: PUSH 00000000
096: POP var_16
098: PUSH var_16
100: PUSH 00000002
102: JEQ 106
104: JMP 216
106: PRINT 08fd9620
108: PRINT 40a3fb0c
[...]
212: PRINT 13784f69
214: EXIT 00000000
216: PRINT 1575002d
218: PRINT 1574450c
[...]
262: PRINT 2fb8c669
264: EXIT 00000001
```

C'est le code de validation. On a une grosse série de `PRINT` à la fin que l'on XOR uniquement avec la valeur obtenue à la dernière machine virtuelle. En effet, le pseudo-assembleur appelle la fonction `PRINT` de la sur-machine avec une variable qui ne fait plus de XOR. Sinon, on prend les deux derniers hex de chaque ligne et on le met dans [CyberChef](https://gchq.github.io/CyberChef/) avec un magic block pour voir comprendre que ça affiche la chaîne `"Congratulations, you won. Validate with FCSC{<input>}"` ou la chaîne `"Noooooo, damn you lost!"`...

#### **Le fameux serial**

Reste à voir les conditions pour valider et afficher la bonne chaîne. On sépare l'entrée en quatre blocs de 4 bytes chacun et on a des condition sur chacun des blocs.

On veut que `var_16` soit égale à 2 donc on doit avoir la fameuse opération `MUL` qui prend une certaine valeure `MUL(input1, 117052c0) == 1`, `MUL(input3, 278bce9d)`

On a quatre autres conditions `input2 MOD 77f3 == 4926`, `input2 MOD 7c49 == 3159`, `input4 MOD 77f3 == 28b2` et `input4 MOD 7c49 == 44a9`. De l'algèbre assez classique, [le théorème des restes chinois](https://fr.wikipedia.org/wiki/Th%C3%A9or%C3%A8me_des_restes_chinois) nous donne une condition sur l'entrée modulo `77f3*7c49`. Pour input2, on obtient `45645430` en oubliant pas qu'on est en little-endian qui donne un résultat parfaitement compréhensible mais ça ne fonctionne pas pour input4. En fait, comme on est modulo `3a3be84b`, on peut l'ajouter au résultat sans invalider les conditions. On obtient alors `33703372` qui donne un résultat acceptable.

#### **MUL** ou comment perdre beaucoup de temps pour quelques lignes de code

Maintenant, on a deux propriétés à valider avec la fonction `MUL`. Dans `Ghidra`, le code obtenu est celui ci :

```c
void MUL(conteneur *container) {
  int iVar1;
  int iVar2;
  ulong uVar3;
  ulong uVar4;
  
  iVar1 = container->stack[container->stack_cptr];
  container->stack_cptr = container->stack_cptr + -1;
  iVar2 = container->stack[container->stack_cptr];
  container->stack_cptr = container->stack_cptr + -1;
  container->stack_cptr = container->stack_cptr + 1;
  uVar4 = (long)iVar1 * (long)iVar2;
  uVar3 = (uVar4 - uVar4 / 0x7fffffff >> 1) + uVar4 / 0x7fffffff >> 0x1e;
  container->stack[container->stack_cptr] = (int)uVar4 - ((int)(uVar3 << 0x1f) - (int)uVar3);
  return;
}
```

Ainsi, on prend la valeur de la pile, on fait des opérations bizarres avec puis on remet ça dans la pile.

J'ai essayé de comprendre tant bien que mal ce que signifiait ces divisions suivies de `>>>` pour pouvoir inverser la fonction mais je n'ai pas compris. J'ai donc utilisé `z3` pour résoudre la série d'équation qu'on avait avec ce code python.

```python
from z3 import *

x = BitVec("x", 32)
var = BitVec("var", 64)
var2 = BitVec("var2", 64)
var3 = BitVec("var3", 64)
var4 = BitVec("var4", 64)
mul = BitVecVal(0x117052c0, 32)
# mul = BitVecVal(0x278bce9d, 32)
s = Solver()
s.add(var == ZeroExt(32,x)*ZeroExt(32,mul))
s.add(var2 == var/0x7fffffff)
s.add(var3 == ((((var-var2)>>1) + var2) >> 0x1e))
s.add(var4 == (var-((var3<<0x1f)-var3)))
s.add(var4 == 0x00000001)
print(s.check())
print(s.model())
```

Il arrive à me trouver un résultat au bout d'une demi-heure avec `x == 0xe54e3376` mais cela n'est pas transformable en caractères imprimables ASCII classique (c'est de l'ASCII étendu). Je met ici cette partie de résolution à titre indicatif, `z3` reste un outil intéressant !!

#### **Le bruteforce**

Quand on ne sait pas quoi faire, on lance un bruteforce. Je l'ai fait avec les caratrères alpha-numérique. Je l'ai fait en python pour une preuve de concept mais ça aurait été plus efficace en `C++` évidemment.

```python
mul_fact_1 = 0x117052c0
mul_fact_2 = 0x278bce9d

letters = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

def find_numb(mul_fact):
    for i in letters:
        for j in letters:
            for k in letters:
                for l in letters:
                    var = int(f"{ord(i):02x}{ord(j):02x}{ord(k):02x}{ord(l):02x}",16)
                    a = var*mul_fact
                    uVar2 = a//0x7fffffff
                    uVar3 = (((a-uVar2)>>1) + uVar2) >> 0x1e
                    uVar4 = a-((uVar3 << 0x1f)-uVar3)
                    if uVar4 < 0:
                        uVar4 += 0x100000000
                    if uVar4 == 0x00000001:
                        return var
        print(i, " done", sep='')

print(hex(find_numb(mul_fact_1)))
print(hex(find_numb(mul_fact_2)))
```

Et on obtient le serial complet avec des lettres de l'alphabet standard !!

![meme_vmv](images/brain_virtual_machine.jpg)