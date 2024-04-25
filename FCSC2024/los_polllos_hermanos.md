# Los Polllos Hermanos
Write-up by *loulous24* **CC-BY-SA 4.0**

***REVERSE*** challenge

## Introduction

Los Polllos Hermanos is a reverse challenge from the 2024 French national contest (FCSC) in the hardware category. This challenge has been solved only 5 times during the contest and I did not solve it at that time. I have spent more time to understand it afterwards and finally solved it.

Let's take a look at the description of the challenge. There is a binary attached.
```
Our experts infiltrated the secret networks of the ‘Los Polllos Hermanos’ gang and managed to retrieve a crucial binary from this cyber underworld organization. Protected by a supposedly invulnerable password, this binary resisted all our extraction attempts. Beware, this organization is known for recruiting mathematicians into its ranks. Can you help us put an end to the activities of ‘Los Polllos Hermanos’?
```

What is interesting here is that it is also a math challenge and that probably [LLL algorithm](https://en.wikipedia.org/wiki/Lenstra%E2%80%93Lenstra%E2%80%93Lov%C3%A1sz_lattice_basis_reduction_algorithm) is involved.

## How to begin

I started by opening the binary with [Ghidra](https://ghidra-sre.org/). There is a main function called by `__libc_start_main` which is

```c
  uint i_16;
  int *errno_ptr;
  int j;
  int k;
  int m;
  int n;
  int iCell;
  int iFinal;
  int local_12c8;
  long len;
  ulong i;
  ulong max;
  uint local_12a8 [28];
  ulong errno_array [256];
  ulong result_mult [256];
  byte local_238 [31];
  char input [257];
  char local_118 [256];
  ulong value;
  
  i = 0;
  setvbuf(stdin,(char *)0x0,_IONBF,0);
  setvbuf(stdout,(char *)0x0,_IONBF,0);
  len = read(STDIN_FILENO,input + 1,0x100);
  if (len == 0) {
                    /* WARNING: Subroutine does not return */
    _exit(1);
  }
  if (input[len] == '\n') {
    input[len] = '\0';
    len = len + -1;
  }
  if (len != 0x100) {
    printf("Error: the input must be 256-byte long [got %lu].\n",len);
                    /* WARNING: Subroutine does not return */
    _exit(1);
  }
  j = 0;
  do {
    /* 14 times doing syscall and errno */
    i_16 = (uint)i & 0xfffffff0;
    errno_array[j] = (ulong)(byte)input[i + 1];
    for (k = 0; k < 0xe; k = k + 1) {
      syscall(errno_array[j]);
      errno_ptr = __errno_location();
      errno_array[j] = (long)*errno_ptr;
    }
    j = j + 1;
    if (j == 0x10) {
      /* Matrix multiplication */
      for (m = 0; m < 0x10; m = m + 1) {
        result_mult[(int)(m + i_16)] = 0;
        for (n = 0; n < 0x10; n = n + 1) {
          result_mult[(int)(m + i_16)] =
               result_mult[(int)(m + i_16)] +
               errno_array[n] *
               (ulong)MATRIX_ARRAY[(long)m + ((long)n + (i & 0xfffffffffffffff0)) * 0x10];
        }
        /* Modulo by 0xfffffffb */
        value = result_mult[(int)(m + i_16)] / 0xfffffffb;
        result_mult[(int)(i_16 + m)] = result_mult[(int)(m + i_16)] - ((value << 0x20) + value * -5)
        ;
      }

      /* Comparison of the maximum with 0xff */
      max = 0;
      for (iCell = 0; iCell < 0x10; iCell = iCell + 1) {
        value = result_mult[(int)(iCell + i_16)];
        if (result_mult[(int)(iCell + i_16)] <= max) {
          value = max;
        }
        max = value;
      }
      if (0xff < max) {
        puts("Error: Wrong password.");
                    /* WARNING: Subroutine does not return */
        _exit(1);
      }
      j = 0;
    }
    i = i + 1;
    if (0xff < i) {
      /* Derivation to get the flag */
      for (iFinal = 0; iFinal < 0x100; iFinal = iFinal + 1) {
        local_118[iFinal] = (char)result_mult[iFinal];
      }
      set_numbers(local_12a8);
      FUN_00101718(local_12a8,(uint *)local_118,0x100);
      FUN_001017d7(local_12a8,local_238);
      puts("Congrats! You can validate the challenge with the flag:");
      printf("FCSC{");
      for (local_12c8 = 0; local_12c8 < 0x20; local_12c8 = local_12c8 + 1) {
        printf("%02x",(ulong)local_238[local_12c8]);
      }
      puts("}");
                    /* WARNING: Subroutine does not return */
      _exit(0);
    }
  } while( true );
```

So `0x100` characters are read on standard input. For each character, the errno of the syscall whose number is the character is called 14 times. Then it is packed by packets of 16 in a vector which is multiplied by a matrix. The flag is displayed if all the results of the matrix multiplication are between 0 and 255.

I have not tried to understand deeply the function responsible for displaying the flag. The code was complicated, and I was sure the goal of the challenge was to solve the part described above.

## Syscall of the previous errno????

During the FCSC, I was stuck because of the code calling syscall and taking as a result the errno and that, 14 times in a row. I did not understand how the binary did not crash with the weird syscalls. I used [GDB](https://fr.wikipedia.org/wiki/GNU_Debugger) to script and see the result of the computation of the loop with this code.

```py
from pwn import ELF, context, process

b = ELF('./los-polllos-hermanos')
context.binary = b
context.terminal = ["gnome-terminal", "--", "sh", "-c"]
context.log_level = 'WARNING'

def test_bytes(bytes_in, len_test=16):
    with open('test_input.data', 'wb') as f:
        f.write(bytes_in)

    gdb_args = ["gdb", "--batch", "./los-polllos-hermanos", "-ex", "set args < ./test_input.data", "-ex", "b *0x555555556308",
                  "-ex", "run"]
    for i in range(len_test):
        gdb_args.append("-ex")
        gdb_args.append(f"x/xg $rbp-0x{0x1230-8*i:04x}")
        gdb_args.append("-ex")
        gdb_args.append("continue")
    io = process(gdb_args)
    
    ret = None
    for i in range(len_test):
        io.readuntil(b'Breakpoint 1, 0x0000555555556308 in ?? ()\n')
        rep = io.readline(keepends=False)
        addr, value = map(lambda x: int(x[2:], 16), rep.split(b':\t'))
        if ret is None:
            ret = value
        print(rep.decode())

    io.close()
    return ret
```

It writes the willed input inside the file `test_input.data`. Then, it executes `gdb` in `--batch` mode and see the content of `errno_array[i]`. However, the result given is only `0xe` or `0x16`.

It's because, I have not seen the array `__DT_INIT_ARRAY` and the function `_INIT_1` at `0x01ad1` during the CTF :'(

There is another function listed in `__DT_INIT_ARRAY`. [This blogpost](https://maskray.me/blog/2021-11-07-init-ctors-init-array) explains more in details how it works. The functions listed in this array are executed before the `main` function. So another function is executed before `main`.

## Seccomp

Inside this function `_INIT_1`, the following code is executed at the beginning.

```c
  int result_prctl;
  long result_call;                    
  sock_fprog seccomp_fprog;
  sock_filter seccomp_filter_0;

  result_prctl = prctl(PR_SET_NO_NEW_PRIVS,1,0,0,0);
  if (result_prctl == -1) {
                    /* WARNING: Subroutine does not return */
    _exit(1);
  }
  seccomp_filter_0.code = 6;
  seccomp_filter_0.jt = '\0';
  seccomp_filter_0.jf = '\0';
  seccomp_filter_0.k = 0x7fff0000;
  seccomp_fprog.len = 1;
  seccomp_fprog.filter = &seccomp_filter_0;
  lVar2 = syscall(SECCOMP,SECCOMP_SET_MODE_FILTER,0,&seccomp_fprog);
  if (result_call == -1) {
                    /* WARNING: Subroutine does not return */
    _exit(1);
  }
  result_call = ptrace(PTRACE_TRACEME,0,1,0);
  if (result_call != -1) {
  	/* Intersting code loading seccomp rules here */
	/* [...] */
  }
```

This explains well the result with GDB. The line `ptrace(PTRACE_TRACEME,0,1,0)` is here to check if the process has an observer which is the case when using `gdb`. And so the code inside the `if` block is not executed.

### Seccomp Tools

It also explains why when using [Seccomp Tools](https://github.com/david942j/seccomp-tools) with `seccomp-tools dump` to get the seccomp filter, the result is only `return ALLOW` because `seccomp-tools` also traces the binary and is detected. The other seccomp rules are not loaded and so not analysed.

### Seccomp rules

Inside the `if` block, the loading of the seccomp rules follows this pattern.

```c
    memcpy(asStack_31398,sock_filter_ARRAY_00103120,0x3840);
    auStack_31488.len = 0x708;
    auStack_31488.filter = asStack_31398;
    result_call = syscall(SECCOMP,SECCOMP_SET_MODE_FILTER,0,&auStack_31488);
    if (result_call == -1) {
                    /* WARNING: Subroutine does not return */
      _exit(1);
    }
```

The content of sock_filter_ARRAY is copied and a call to seccomp is made with syscall.

This is an extract of the result of the output of `seccomp-tools disasm`.

```
 line  CODE  JT   JF      K
=================================
 0000: 0x20 0x00 0x00 0x00000000  A = sys_number
 0001: 0x15 0x04 0x00 0x00000000  if (A == read) goto 0006
 0002: 0x15 0x03 0x00 0x00000001  if (A == write) goto 0006
 0003: 0x15 0x02 0x00 0x0000003b  if (A == execve) goto 0006
 0004: 0x15 0x01 0x00 0x000000e7  if (A == exit_group) goto 0006
 0005: 0x15 0x00 0x01 0x0000013d  if (A != seccomp) goto 0007
 0006: 0x06 0x00 0x00 0x7fff0000  return ALLOW
 0007: 0x20 0x00 0x00 0x00000000  A = sys_number
 0008: 0x01 0x00 0x00 0x43f8618b  X = 1140351371
 0009: 0x2c 0x00 0x00 0x00000000  A *= X
 0010: 0x01 0x00 0x00 0x50ce4fef  X = 1355698159
 0011: 0x0c 0x00 0x00 0x00000000  A += X
 0012: 0x15 0x00 0x01 0x7cb0167b  if (A != 2091914875) goto 0014
 0013: 0x06 0x00 0x00 0x0005005a  return ERRNO(90)
 0014: 0x20 0x00 0x00 0x00000000  A = sys_number
 0015: 0x01 0x00 0x00 0xbe26ff8b  X = 3190226827
 0016: 0x2c 0x00 0x00 0x00000000  A *= X
 0017: 0x01 0x00 0x00 0x2c2380b1  X = 740524209
 0018: 0x0c 0x00 0x00 0x00000000  A += X
 0019: 0x15 0x00 0x01 0xaa0fdcde  if (A != 2853166302) goto 0021
 0020: 0x06 0x00 0x00 0x00050b9b  return ERRNO(2971)
 [...]
```

So except for syscalls `read`, `write`, `execve`, `exit_group` and `seccomp`, the syscall number is tested with some constants and a specific errno is returned depending on the syscall. This is why the program is not crashing when executing wrong syscalls because it answers with errno.

It is possible to dump the array of the seccomp rules and use `seccomp-tools disasm` to see the rules, but there are 26000 rules. I have chosen to get the array for translating each character of input to the bytes used for the multiplication with the matrix. I used this code (almost the same but note the added breakpoint).

A temporary breakpoint is set at the address `0x01ba7` to change the value of `$rax` before the comparison after the call to `ptrace`. It is here to execute the code defining the seccomp rules even if the process is traced by `gdb`.

```py
def test_bytes(bytes_in, len_test=16):
    with open('test_input.data', 'wb') as f:
        f.write(bytes_in)

    gdb_args = ["gdb", "--batch", "./los-polllos-hermanos", "-ex", "set args < ./test_input.data", "-ex", "b *0x555555556308", "-ex", "tb *0x555555555ba7",
                  "-ex", "run", "-ex", "set $rax = 0"]
    for i in range(len_test):
        gdb_args.append("-ex")
        gdb_args.append("continue")
        gdb_args.append("-ex")
        gdb_args.append(f"x/xg $rbp-0x{0x1230-8*i:04x}")
    io = process(gdb_args)
    
    ret = None
    for i in range(len_test):
        io.readuntil(b'Breakpoint 1, 0x0000555555556308 in ?? ()\n')
        rep = io.readline(keepends=False)
        addr, value = map(lambda x: int(x[2:], 16), rep.split(b':\t'))
        if ret is None:
            ret = value
        print(rep.decode())

    io.close()
    return ret
```

And this code call the previous function for each byte and define a dictionary for translating the input to the bytes being multiplied by the matrix.

```py
translation = {}
for byte in range(256):
    if byte == 231:
        continue
    print(f"Testing byte {byte}")
    bytes_in = [byte]*256
    if bytes_in[-1] == 10:
        bytes_in[-1] = 0
    res = test_bytes(bytes(bytes_in), len_test=2)
    translation[byte] = res

print(translation)
```

The translation dictionary is the following (some syscalls errno are set above 256, those have been removed here)

```py
translation = {2: 167, 3: 145, 4: 190, 5: 235, 6: 170, 7: 91, 9: 204, 10: 237, 11: 67, 12: 229, 13: 199, 14: 48, 15: 106, 16: 159, 17: 182, 18: 241, 19: 38, 20: 236, 21: 147, 22: 171, 23: 141, 24: 213, 25: 40, 26: 83, 27: 49, 28: 62, 29: 123, 30: 174, 31: 75, 32: 25, 33: 47, 34: 188, 35: 53, 36: 34, 37: 248, 38: 100, 39: 158, 40: 81, 41: 194, 42: 180, 43: 189, 44: 129, 45: 36, 46: 240, 47: 16, 48: 120, 49: 6, 50: 150, 51: 4, 52: 88, 53: 156, 54: 225, 55: 233, 56: 222, 57: 66, 58: 97, 60: 220, 61: 42, 62: 209, 63: 137, 64: 176, 65: 247, 66: 18, 67: 110, 68: 201, 69: 132, 70: 197, 71: 24, 72: 172, 73: 35, 74: 206, 75: 23, 76: 148, 77: 121, 78: 161, 79: 86, 80: 122, 81: 117, 82: 157, 83: 244, 84: 101, 85: 82, 86: 76, 87: 210, 88: 133, 89: 68, 90: 61, 91: 177, 92: 162, 93: 8, 94: 99, 95: 184, 96: 65, 97: 70, 98: 231, 99: 104, 100: 78, 101: 207, 102: 119, 103: 127, 104: 155, 105: 163, 106: 59, 107: 3, 108: 195, 109: 160, 110: 10, 111: 50, 112: 64, 113: 20, 114: 103, 115: 138, 116: 95, 117: 151, 118: 183, 119: 21, 120: 242, 121: 60, 122: 72, 123: 198, 124: 228, 125: 51, 126: 80, 127: 202, 129: 90, 130: 149, 131: 185, 132: 245, 133: 22, 134: 17, 135: 142, 136: 144, 137: 30, 138: 223, 139: 181, 140: 52, 141: 31, 142: 216, 143: 179, 144: 102, 145: 168, 146: 73, 147: 253, 148: 19, 149: 146, 150: 109, 151: 32, 152: 200, 153: 14, 154: 115, 155: 187, 156: 254, 157: 92, 158: 12, 159: 85, 160: 74, 161: 215, 162: 9, 163: 191, 164: 71, 165: 33, 166: 140, 167: 15, 168: 58, 169: 46, 170: 125, 171: 136, 172: 243, 173: 11, 174: 250, 175: 41, 176: 251, 177: 166, 178: 27, 179: 239, 180: 128, 181: 186, 182: 124, 183: 165, 184: 249, 185: 230, 186: 152, 187: 227, 188: 44, 189: 208, 190: 107, 191: 26, 192: 39, 193: 224, 194: 55, 195: 45, 196: 232, 197: 84, 198: 28, 199: 175, 200: 1, 201: 139, 202: 94, 203: 2, 204: 131, 205: 252, 206: 63, 207: 96, 208: 173, 209: 87, 210: 193, 211: 126, 212: 214, 213: 169, 214: 112, 215: 246, 216: 226, 217: 134, 218: 98, 219: 56, 220: 211, 221: 118, 222: 116, 223: 234, 224: 218, 225: 69, 226: 130, 227: 13, 228: 37, 229: 79, 230: 89, 232: 108, 233: 192, 234: 57, 235: 221, 236: 135, 237: 255, 238: 43, 239: 5, 240: 111, 241: 196, 242: 212, 243: 7, 244: 143, 245: 205, 246: 114, 247: 219, 248: 154, 249: 105, 250: 113, 251: 93, 252: 217, 253: 238, 254: 54, 255: 203}
```

## Lenstra–Lenstra–Lovász

So now, there is a vector of translated input multiplied by a matrix of constant values. The code 
```c
value = result_mult[(int)(m + i_16)] / 0xfffffffb;
result_mult[(int)(i_16 + m)] = result_mult[(int)(m + i_16)] - ((value << 0x20) + value * -5);
```
is doing a modulo by `0xfffffffb` for each cell of the vector after multiplication.

For each block $i$ of 16 bytes, the objective is to find $(v_{0}^i, v_{1}^i, \dots, v_{15}^i)$ between 0 and 255 such that

$$
\left(\begin{matrix}
m_{0,0}^i & m_{1,0}^i & \dots & m_{15,0}^i \\
m_{0,1}^i & m_{1,1}^i & \dots & m_{15,1}^i \\
\vdots & \vdots & \ddots & \vdots \\
m_{0,15}^i & m_{1,15}^i & \dots & m_{15,15}^i
\end{matrix}\right)

\left(\begin{matrix}
v_{0}^i\\
v_{1}^i\\
\vdots \\
v_{15}^i
\end{matrix}\right) = 

\left(\begin{matrix}
x_{0}^i + k_{0}^i\texttt{0xfffffffb}\\
x_{1}^i + k_{1}^i\texttt{0xfffffffb}\\
\vdots \\
x_{15}^i + k_{15}^i\texttt{0xfffffffb}
\end{matrix}\right)
$$
with $\left(x_j^i\right)_{j \in \llbracket 0, \dots, 15\rrbracket}$ values between 0 and 255 and $\left(k_j^i\right)_{j \in \llbracket 0, \dots, 15\rrbracket}$ integers.

It is a [Closest Vector Problem](https://en.wikipedia.org/wiki/Lattice_problem) (CVP).


### Lattice theory

A set of vector (the basis) are added or subtracted together several times to form a lattice. The goal of the LLL algorithm is to try to find a basis, a set of vector generating the same lattice, with small vectors.

So the result of the LLL algorithm will be another basis. As each vector of the new basis belongs to the lattice, it is also a linear combination of the first set of vector. So LLL is returning vectors being linear combination of the input basis and LLL tries to make them small.

Once that said, all the game is to constraint LLL enough to produce a basis with one vector that is small enough to validate the constraint of the challenge.

For the rest of this challenge, the basis will be represented by a matrix and each column is a vector of the basis.

### LLL in this context

#### Knowing the result

There are several tricks performed here, the first one is to know how to get the number of times a vector has been taken (in order to know what is the input). This can be done by appending an identity matrix below the first matrix.

$$
\left(\begin{matrix}
m_{0,0}^i & m_{1,0}^i & \dots & m_{15,0}^i \\
m_{0,1}^i & m_{1,1}^i & \dots & m_{15,1}^i \\
\vdots & \vdots & \ddots & \vdots \\
m_{0,15}^i & m_{1,15}^i & \dots & m_{15,15}^i \\
1 & 0 & \dots & 0 \\
0 & 1 & \dots & 0 \\
\vdots & \vdots & \ddots & \vdots \\
0 & 0 & \dots & 1 \\
\end{matrix}\right)
$$

When a linear combination of the columns is done the values of rows 16 to are the number of time each column is present. It is important to know by how many it is needed to multiply the column to get the result, it corresponds to $(v_j^i)$. These rows will also be minimised by LLL. It is exactly what is wanted because the $(v_j^i)$ should be small.

#### Modulo operation

The result is taken modulo `0xfffffffb`. There is a trick to mimic a modulo operation with a lattice basis. The trick is to add an identity matrix to right (or to the left) multiplied by the modulo value. Each column of this matrix can be added to the result to reduce it and if the matrix below the matrix added is null, the vectors added during the linear combination will not be counted. So it is possible to add any number of them which corresponds well to the modulo operation.

$$
\left(\begin{matrix}
m_{0,0}^i & m_{1,0}^i & \dots & m_{15,0}^i & \texttt{0xfffffffb} & 0 & \dots & 0 \\
m_{0,1}^i & m_{1,1}^i & \dots & m_{15,1}^i & 0 & \texttt{0xfffffffb} & \dots & 0 \\
\vdots & \vdots & \ddots & \vdots & \vdots & \vdots & \ddots & \vdots \\
m_{0,15}^i & m_{1,15}^i & \dots & m_{15,15}^i & 0 & 0 & \dots & \texttt{0xfffffffb} \\
1 & 0 & \dots & 0 & 0 & 0 & \dots & 0\\
0 & 1 & \dots & 0 & 0 & 0 & \dots & 0 \\
\vdots & \vdots & \ddots & \vdots & \vdots & \vdots & \ddots & \vdots \\
0 & 0 & \dots & 1 & 0 & 0 & \dots & 0 \\
\end{matrix}\right)
$$

### Result column

So the result column with both tricks above done will be with the following form.

$$
\left(\begin{matrix}
x_{0}^i \\
x_{1}^i \\
\vdots \\
x_{15}^i \\
v_{0}^i \\
v_{1}^i \\
\vdots \\
v_{15}^i
\end{matrix}\right)
$$

With $(x_j^i)$ the result between 0 and 255 for each and $(v_j^i)$ the input between 0 and 255 too.

So the goal is to find the closest vector of the lattice near $(128, 128, \dots, 128, 128, 128, \dots, 128)$. A deviation lower than `-128` and `+128` is accepted.

From now, there are two solutions, one using the Babai algorithm, which performs LLL and finds the closest vector or converting the closest vector problem into a smallest vector problem (SVP) and solve it directly with LLL.

### Babai algorithm

I will not go into details here, but it works well with sage and gives a closed vector. In the code below, `l` is a list of matrix which index `i` is holding the ith matrix with the [column-major order](https://en.wikipedia.org/wiki/Row-_and_column-major_order).

```py
from sage.modules.free_module_integer import IntegerLattice
from fpylll import IntegerMatrix, CVP, BKZ

results_babai = []
for l_i in l:
	M = Matrix([l_i[i] + [1 if i == j else 0 for j in range(16)] for i in range(16)] + [[0xfffffffb if j == i else 0 for j in range(16)] + [0]*16 for i in range(16)])
	lattice = IntegerLattice(M, lll_reduce=True)

	target = vector([128]*16 + [128]*16)
	result = lattice(CVP.babai(IntegerMatrix.from_matrix(reseau.reduced_basis), target))

	results_babai.append(result[16:])
	print(result[16:])
```

A lattice is defined and reduce and Babai is called to find the CVP.

### LLL direct reduction

The LLL finds columns that are as small as possible, so it does not fit well with CVP. A trick is to add the opposite of the target as a column and to add a row with a big weight that will check that is column is taken at most one time. As LLL returns a basis, it is assured that at least for one vector, this line will not be zero. The goal is to have this vector equals to the willed result.

$$
\left(\begin{matrix}
WEIGHT & 0 & 0 & \dots & 0 & 0 & 0 & \dots & 0 \\
-128 & m_{0,0}^i & m_{1,0}^i & \dots & m_{15,0}^i & \texttt{0xfffffffb} & 0 & \dots & 0 \\
-128 & m_{0,1}^i & m_{1,1}^i & \dots & m_{15,1}^i & 0 & \texttt{0xfffffffb} & \dots & 0 \\
-128 & \vdots & \vdots & \ddots & \vdots & \vdots & \vdots & \ddots & \vdots \\
-128 & m_{0,15}^i & m_{1,15}^i & \dots & m_{15,15}^i & 0 & 0 & \dots & \texttt{0xfffffffb} \\
-128 & 1 & 0 & \dots & 0 & 0 & 0 & \dots & 0\\
-128 & 0 & 1 & \dots & 0 & 0 & 0 & \dots & 0 \\
-128 & \vdots & \vdots & \ddots & \vdots & \vdots & \vdots & \ddots & \vdots \\
-128 & 0 & 0 & \dots & 1 & 0 & 0 & \dots & 0 \\
\end{matrix}\right)
$$

The result is expected to be between -128 and 128 for each row and the first row should be $WEIGHT$ or $-WEIGHT$ (and the result is the opposite).

Here the columns 17 to 33 hold the input making the result of the multiplication low.

```py
results_LLL = []
for l_i in l:
	weight = 128 << 7
	MLLL = Matrix([
    [weight] + [-128]*16 + [-128]*16] + # first column with the goal vector
    [[0] + [l_i[i][j] for j in range(16)] + [1 if i == j else 0 for j in range(16)] for i in range(16)] + # columns with the mi
    [[0] + [(1<<32)-5 if j == i else 0 for j in range(16)] + [0]*16 for i in range(16)]) # columns for the modulo

	MLLL = MLLL.LLL()
	MLLL[:,0] /= weight
	offset_line = Matrix([[128]*32])

  # Find the vector veryfing the willed properties
	for i in range(33):
		MLLL[i,1:] += offset_line
		if MLLL[i,0] < 0:
			MLLL[i,:] *= -1
		if MLLL[i,0] != 1:
			continue
		good_result = True
		for j in range(1,1+16*2):
			if MLLL[i,j] < 0 or MLLL[i,j] >= 0x100:
				good_result = False
				break
		if not good_result:
			continue
		print(vector(MLLL[i,17:]))
		results_LLL.append(vector(MLLL[i,17:]))
		break

```

## Getting the flag

So now, the vectors before the multiplication are found. The last step is to convert them again to the input and run the binary to execute the last step.

```py
translation_inv = {}
for k, v in translation.items():
    translation_inv[v] = k

translation_inv = {}
for k, v in translation.items():
    translation_inv[v%256] = k

password = bytearray()
for i in range(len(results_LLL)):
    for x in results_LLL[i]:
        password.append(translation_inv[x])

io = process('./los-polllos-hermanos')
io.write(password)
io.readuntil(b'You can validate the challenge with the flag:\n')
flag = io.readlineS(keepends=False)

print(flag)
```
