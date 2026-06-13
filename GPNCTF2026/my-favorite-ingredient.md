## \[0x00\] TL;DR

This challenge was a reversing problem with a single 64-bit ELF binary.

The program takes a flag as an argument, and if the internal verification routine succeeds, it prints `Correct flag!`.

The final flag is:
```text
GPNCTF{jus7_ON3_ONStRUCTion5_1s_aL1_Y0U_n3eD_MAY8E1239794FkfNDh}
```

---

## \[0x01\] File Analysis

After extracting the archive and opening the binary in a decompiler, I noticed that the symbols were not stripped. Because of that, the function names were still visible, which made the initial analysis much easier.

![|200](https://i.imgur.com/EBJp74Q.png)

To get a rough idea of how the program behaves, I first executed it without any arguments.
```bash
./my-favorite-ingredient
```

```text
Usage: ./my-favorite-ingredient <flag>
```

Then I tried passing a short string.
```bash
./my-favorite-ingredient test
```

```text
Flag must be 64 characters long.
```

From these outputs, I could tell that the flag must be exactly 64 bytes long.

After that, I moved on to the decompiler and looked at the overall logic. The functions that stood out were:
```text
main
verify_flag
matvec_mul_vectorized
matvec_mul_bitslice
```

Following the call flow from `main`, the structure looked roughly like this:
```text
main
 └── verify_flag
      └── matvec_mul_vectorized
           └── matvec_mul_bitslice
```

In other words, the program seems to treat the input flag as a vector, multiply it by an internal matrix, and compare the result with a specific target value.

## \[0x02\] main Analysis

The code in `main` looked like this:

![](https://i.imgur.com/kXhoEw2.png)
Here, `dest` is `0x1000 = 4096` bytes.
Since `4096 = 64 * 64`, it naturally suggests a 64×64 matrix.
Also, `v8` is 64 bytes long.

So at this point, I assumed that the overall verification would look roughly like this:
```
matrix * input == target
```

Of course, this was only an initial guess. The actual verification routine has a few extra transformations before the comparison.

## \[0x03\] verify_flag Analysis

At the beginning of `verify_flag`, the input is transformed byte by byte using AVX instructions.

![|300](https://i.imgur.com/bAoWngY.png)
The important instructions were:
```asm
vpmullw
vpand
vpackuswb
vpaddb
```

This transformation is equivalent to the following operation for each input byte:
```text
x = input * 197 + 0x65 mod 256
```

After this transformation, the 64-byte input is passed to `matvec_mul_vectorized`.
```asm
1204: call matvec_mul_vectorized
```

Then the output produced by `matvec_mul_vectorized` is compared with `v8`.

The comparison part looked like this:
```asm
1209: mov    cl, BYTE PTR [rbx]
120b: not    cl
120f: cmp    BYTE PTR [rsp], cl
```

Here, `rbx` points to `v8`.

This means the output is not compared directly with `v8`, but with the bitwise NOT of `v8`.

The structure can be summarized as:
```text
output = matvec(transformed_input, dest)

for i in range(64):
    if output[i] != ~v8[i]:
        return false
```

At first, the input transformation inside `verify_flag` made the equation look more complicated.
```text
x = 197 * c + 0x65 mod 256
```

However, after looking deeper into `matvec_mul_vectorized`, I found that another transformation is applied to the input.
```text
y = 13 * x + 0xdf mod 256
```

Combining the two equations gives:
```text
y = 13 * (197 * c + 0x65) + 0xdf mod 256
```

Calculating this:
```text
13 * 197 = 2561
13 * 0x65 + 0xdf = 13 * 101 + 223 = 1536
```

So:
```text
y = 2561 * c + 1536 mod 256
```

Since:
```text
2561 ≡ 1 mod 256
1536 ≡ 0 mod 256
```

The result becomes:
```text
y ≡ c mod 256
```

In other words, the two transformations cancel each other out.

The actual flow is:
```
input c
→ transformed in verify_flag as 197*c + 0x65
→ transformed again in matvec_mul_vectorized as 13*x + 0xdf
→ returns back to c
```

So the actual input that enters the matrix operation can be treated as the original flag bytes.

## \[0x04\] matvec_mul_bitslice Analysis

Inside `matvec_mul_vectorized`, the function `matvec_mul_bitslice` is called.

As its name suggests, this function is implemented in a bitsliced style. It is also quite long, and the decompiler could not recover it cleanly.

There were many AVX instructions and repeated bitwise operations, so I decided that fully reconstructing the function statically would take too much time.

At first, it looked like a matrix multiplication over GF(2).

In other words, I suspected a structure like this:
```
output_bit = XOR(input_bit & matrix_bit)
```

However, after testing several inputs dynamically and comparing the outputs, the function did not behave like an XOR-linear function. Instead, it behaved more like a byte-wise additive linear function.

So the function could be modeled as:
```
F(input) = A * input + b mod 256
```

Here, `A` is a 64×64 matrix, and `b` is a constant vector.

Once I had this model, I no longer needed to fully reverse `matvec_mul_bitslice`. Instead, I treated the function as a black box and collected outputs while changing the input.

## \[0x05\] Dynamic Analysis
Looking at the code right after the call to `matvec_mul_vectorized` inside `verify_flag`, we can see the following:
```
1204: call matvec_mul_vectorized
1209: mov cl, BYTE PTR [rbx]
120b: not cl
120f: cmp BYTE PTR [rsp], cl
```

The important point here is that the result of `matvec_mul_vectorized` is stored on the stack at `rsp`.

So if execution stops at `0x1209`, the comparison has not started yet, and the 64-byte output vector is still available on the stack.

Therefore, I can set a breakpoint at this location and dump 64 bytes from `rsp`.

In GDB, it would look like this:
```
b *verify_flag+0x99
run AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
x/64bx $rsp
```

Since the binary is PIE, the actual runtime address needs to be calculated by adding the PIE base address.

---

## [0x06] Matrix Recovery

Assume the verification function has the following form:
```
F(x) = A * x + b mod 256
```

What we want is:
```
F(flag) = ~v8
```

Even if we do not know `A` and `b` directly, we can recover the matrix as long as we can execute `F`.

First, I chose a base input:
```
base = 0x01 * 64
```

Then I changed one position at a time:
```
base[j] = 0x02
```

The output difference becomes:
```
F(base with byte[j] = 2) - F(base) mod 256
```

Since `base[j]` increased from `1` to `2`, this difference corresponds to the `j`-th column of matrix `A`.

Repeating this process 64 times gives the full 64×64 matrix.

The process can be summarized as:
```
1. Compute F(base).
2. For j = 0..63:
   - Change only the j-th byte of base to 2.
   - Compute F(modified).
   - Compute F(modified) - F(base).
   - This becomes the j-th column of A.
```

This approach treats the function as a black box from 64 input bytes to 64 output bytes, and only relies on its linearity.

---

## [0x07] Final Calculation

`verify_flag` does not compare the result directly with `v8`. Instead, it compares the result with the bitwise NOT of `v8`.
```
mov cl, [target+i]
not cl
cmp [output+i], cl
```

So the actual target vector is:
```
goal[i] = (~v8[i]) & 0xff
```

We need to solve:
```
F(flag) = goal
```

Since:
```
F(x) = A * x + b
```

And for the base input:
```
F(base) = A * base + b
```

Subtracting the two equations removes the constant vector `b`.
```
F(flag) - F(base) = A * (flag - base)
```

Therefore, the equation to solve is:
```
A * delta = goal - F(base) mod 256
```

Finally:
```
flag = base + delta mod 256
```

One important detail is that this is not normal integer arithmetic. All operations are performed modulo 256.
```
mod 256
```

So every value is treated as a byte between 0 and 255.

To perform Gaussian elimination, the pivot needs to have a multiplicative inverse.

In `mod 256`, a value has an inverse only when it is odd.

Fortunately, the recovered matrix was invertible, and I was able to choose odd pivots throughout the elimination process.

The elimination logic can be described as:
```
MOD = 256

def inv_mod_256(x):
    return pow(x, -1, MOD)

for col in range(64):
    pivot = find_row_with_odd_value(col)
    swap_rows(col, pivot)

    inv = inv_mod_256(A[col][col])
    normalize_row(col, inv)

    eliminate_other_rows(col)
```

After solving for `delta`, I added it back to `base` and recovered the flag:
```
GPNCTF{jus7_ON3_ONStRUCTion5_1s_aL1_Y0U_n3eD_MAY8E1239794FkfNDh}
```

---

## \[0x08\] PoC

```python
#!/usr/bin/env python3
import os
import struct
import ctypes
import ctypes.util
import subprocess
from pathlib import Path

BIN = Path("./my-favorite-ingredient")
PATCHED = Path("./my-favorite-ingredient.patched")

FLAG_LEN = 64
BREAK_FILE_OFF = 0x1209
TARGET_FILE_OFF = 0x32170
MOD = 256

libc = ctypes.CDLL(ctypes.util.find_library("c"), use_errno=True)
libc.ptrace.restype = ctypes.c_long

PTRACE_TRACEME = 0
PTRACE_PEEKDATA = 2
PTRACE_GETREGS = 12
PTRACE_CONT = 7


class Regs(ctypes.Structure):
    _fields_ = [
        ("r15", ctypes.c_ulonglong), ("r14", ctypes.c_ulonglong),
        ("r13", ctypes.c_ulonglong), ("r12", ctypes.c_ulonglong),
        ("rbp", ctypes.c_ulonglong), ("rbx", ctypes.c_ulonglong),
        ("r11", ctypes.c_ulonglong), ("r10", ctypes.c_ulonglong),
        ("r9", ctypes.c_ulonglong), ("r8", ctypes.c_ulonglong),
        ("rax", ctypes.c_ulonglong), ("rcx", ctypes.c_ulonglong),
        ("rdx", ctypes.c_ulonglong), ("rsi", ctypes.c_ulonglong),
        ("rdi", ctypes.c_ulonglong), ("orig_rax", ctypes.c_ulonglong),
        ("rip", ctypes.c_ulonglong), ("cs", ctypes.c_ulonglong),
        ("eflags", ctypes.c_ulonglong), ("rsp", ctypes.c_ulonglong),
        ("ss", ctypes.c_ulonglong), ("fs_base", ctypes.c_ulonglong),
        ("gs_base", ctypes.c_ulonglong), ("ds", ctypes.c_ulonglong),
        ("es", ctypes.c_ulonglong), ("fs", ctypes.c_ulonglong),
        ("gs", ctypes.c_ulonglong),
    ]


def patch_binary():
    data = bytearray(BIN.read_bytes())

    if data[BREAK_FILE_OFF] != 0xCC:
        assert data[BREAK_FILE_OFF] == 0x8A
        data[BREAK_FILE_OFF] = 0xCC

    PATCHED.write_bytes(data)
    PATCHED.chmod(0o755)


def check(ret, name):
    if ret == -1:
        raise OSError(ctypes.get_errno(), name)


def dump_vector(arg):
    assert len(arg) == FLAG_LEN
    assert b"\x00" not in arg

    pid = os.fork()

    if pid == 0:
        libc.ptrace(PTRACE_TRACEME, 0, None, None)
        os.execve(
            str(PATCHED).encode(),
            [str(PATCHED).encode(), arg],
            os.environ.copy()
        )

    os.waitpid(pid, 0)
    check(libc.ptrace(PTRACE_CONT, pid, None, None), "PTRACE_CONT")

    status = os.waitpid(pid, 0)[1]
    if not os.WIFSTOPPED(status):
        raise RuntimeError("process did not stop")

    regs = Regs()
    check(
        libc.ptrace(PTRACE_GETREGS, pid, None, ctypes.byref(regs)),
        "PTRACE_GETREGS"
    )

    out = bytearray()

    for offset in range(0, FLAG_LEN, 8):
        ctypes.set_errno(0)
        word = libc.ptrace(
            PTRACE_PEEKDATA,
            pid,
            ctypes.c_void_p(regs.rsp + offset),
            None
        )

        if word == -1 and ctypes.get_errno() != 0:
            raise OSError(ctypes.get_errno(), "PTRACE_PEEKDATA")

        out += struct.pack("<Q", word & 0xffffffffffffffff)

    os.kill(pid, 9)

    try:
        os.waitpid(pid, 0)
    except ChildProcessError:
        pass

    return bytes(out[:FLAG_LEN])


def solve_linear(A, b):
    n = len(A)
    mat = [A[i][:] + [b[i]] for i in range(n)]
    row = 0
    pivots = []

    for col in range(n):
        pivot = None

        for i in range(row, n):
            if mat[i][col] & 1:
                pivot = i
                break

        if pivot is None:
            continue

        mat[row], mat[pivot] = mat[pivot], mat[row]

        inv = pow(mat[row][col], -1, MOD)
        mat[row] = [(x * inv) & 0xff for x in mat[row]]

        for i in range(n):
            if i == row:
                continue

            factor = mat[i][col]
            if factor == 0:
                continue

            mat[i] = [
                (mat[i][j] - factor * mat[row][j]) & 0xff
                for j in range(n + 1)
            ]

        pivots.append(col)
        row += 1

    if row != n:
        raise RuntimeError("matrix is not invertible")

    x = [0] * n

    for i, col in enumerate(pivots):
        x[col] = mat[i][n]

    return x


def main():
    patch_binary()

    data = BIN.read_bytes()
    target = data[TARGET_FILE_OFF:TARGET_FILE_OFF + FLAG_LEN]
    goal = bytes((~x) & 0xff for x in target)

    base = bytes([1] * FLAG_LEN)
    base_out = dump_vector(base)

    columns = []

    for j in range(FLAG_LEN):
        modified = bytearray(base)
        modified[j] = 2

        out = dump_vector(bytes(modified))
        column = [(out[i] - base_out[i]) & 0xff for i in range(FLAG_LEN)]
        columns.append(column)

    A = [
        [columns[j][i] for j in range(FLAG_LEN)]
        for i in range(FLAG_LEN)
    ]

    rhs = [(goal[i] - base_out[i]) & 0xff for i in range(FLAG_LEN)]
    delta = solve_linear(A, rhs)

    flag = bytes((base[i] + delta[i]) & 0xff for i in range(FLAG_LEN))

    subprocess.check_output([str(BIN.resolve()), flag])
    print(flag.decode())


if __name__ == "__main__":
    main()
```