# 虛擬機設計

一般會用 swtich case，範例如下：


```cpp
#define OP_HALT     0x0
#define OP_INC      0x1
#define OP_DEC      0x2
#define OP_MUL2     0x3
#define OP_DIV2     0x4
#define OP_ADD7     0x5
#define OP_NEG      0x6

int interp_switch(unsigned char* code, int initval) {
    int pc = 0;
    int val = initval;

    while (1) {
        switch (code[pc++]) {
            case OP_HALT:
                return val;
            case OP_INC:
                val++;
                break;
            case OP_DEC:
                val--;
                break;
            case OP_MUL2:
                val *= 2;
                break;
            case OP_DIV2:
                val /= 2;
                break;
            case OP_ADD7:
                val += 7;
                break;
            case OP_NEG:
                val = -val;
                break;
            default:
                return val;
        }
    }
}
```


用 computed goto 會更快，如下：

```cpp
int interp_cgoto(unsigned char* code, int initval) {
    /* The indices of labels in the dispatch_table are the relevant opcodes
    */
    static void* dispatch_table[] = {
        &&do_halt, &&do_inc, &&do_dec, &&do_mul2,
        &&do_div2, &&do_add7, &&do_neg};
    #define DISPATCH() goto *dispatch_table[code[pc++]]

    int pc = 0;
    int val = initval;

    DISPATCH();
    while (1) {
        do_halt:
            return val;
        do_inc:
            val++;
            DISPATCH();
        do_dec:
            val--;
            DISPATCH();
        do_mul2:
            val *= 2;
            DISPATCH();
        do_div2:
            val /= 2;
            DISPATCH();
        do_add7:
            val += 7;
            DISPATCH();
        do_neg:
            val = -val;
            DISPATCH();
    }
}
```

## Benchmarking

I did some simple benchmarking with random opcodes and the goto version is 25% faster than the switch version. This, naturally, depends on the data and so the results can differ for real-world programs.

Comments inside the CPython implementation note that using computed goto made the Python VM 15-20% faster, which is also consistent with other numbers I've seen mentioned online.

原因是因為

Further down in the post you'll find two "bonus" sections that contain annotated disassembly of the two functions shown above, compiled at the -O3 optimization level with GCC. It's there for the real low-level buffs among my readers, and as a future reference for myself. Here I aim to explain why the computed goto code is faster at a bit of a higher level, so if you feel there are not enough details, go over the disassembly in the bonus sections.

The computed goto version is faster because of two reasons:

1. The switch does a bit more per iteration because of bounds checking.
2. The effects of hardware branch prediction.

