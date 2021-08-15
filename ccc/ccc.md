# rv32emu-next 的主程式


main.c

```cpp
static void run(struct riscv_t *rv)
{
    const uint32_t cycles_per_step = 100;
    for (; !rv_has_halted(rv);) { /* run until the flag is done */
        /* step instructions */
        rv_step(rv, cycles_per_step);
    }
}
```

riscv.c

```cpp
void rv_step(struct riscv_t *rv, int32_t cycles)
{
    assert(rv);
    const uint64_t cycles_target = rv->csr_cycle + cycles;
    uint32_t inst, index;

#define OP_UNIMP op_unimp
#ifdef ENABLE_COMPUTED_GOTO // ccc: 這個版本比較快，為何？
#define OP(instr) &&op_##instr
#define TABLE_TYPE const void *
#else  // ENABLE_COMPUTED_GOTO = false
#define OP(instr) op_##instr
#define TABLE_TYPE const opcode_t
#endif

    // clang-format off
    TABLE_TYPE jump_table[] = {
    //  000         001           010        011           100         101        110   111
        OP(load),   OP(load_fp),  OP(unimp), OP(misc_mem), OP(op_imm), OP(auipc), OP(unimp), OP(unimp), // 00
        OP(store),  OP(store_fp), OP(unimp), OP(amo),      OP(op),     OP(lui),   OP(unimp), OP(unimp), // 01
        OP(madd),   OP(msub),     OP(nmsub), OP(nmadd),    OP(fp),     OP(unimp), OP(unimp), OP(unimp), // 10
        OP(branch), OP(jalr),     OP(unimp), OP(jal),      OP(system), OP(unimp), OP(unimp), OP(unimp), // 11
    };
// clang-format on

#ifdef ENABLE_COMPUTED_GOTO
#define DISPATCH()                                      \
    {                                                   \
        if (rv->csr_cycle >= cycles_target || rv->halt) \
            goto quit;                                  \
        /* fetch the next instruction */                \
        inst = rv->io.mem_ifetch(rv, rv->PC);           \
        /* standard uncompressed instruction */         \
        if ((inst & 3) == 3) {                          \ // 非壓縮格式
            index = (inst & INST_6_2) >> 2;             \
            goto *jump_table[index];                    \ // 根據表格跳躍
        } else {                                        \
            /* TODO: compressed instruction*/           \
            assert(!"Unreachable");                     \
        }                                               \
    }

#define EXEC(instr)                   \
    {                                 \
        /* dispatch this opcode */    \
        if (!op_##instr(rv, inst))    \
            goto quit;                \
        /* increment the cycles csr*/ \
        rv->csr_cycle++;              \
    }

#define TARGET(instr)         \
    op_##instr : EXEC(instr); \
    DISPATCH();

    DISPATCH();

    // main loop // 以下這些是會改變 PC 的，其他不用處理
    TARGET(load) // op##load: EXEC(load); DISPATCH(); // EXEC() 和 DISPATCH() 是上面的 define
    TARGET(op_imm)
    TARGET(auipc)
    TARGET(store)
    TARGET(op)
    TARGET(lui)
    TARGET(branch)
    TARGET(jalr)
    TARGET(jal)
    TARGET(system)
#ifdef ENABLE_Zifencei
    TARGET(misc_mem)
#endif
#ifdef ENABLE_RV32A
    TARGET(amo)
#endif
    TARGET(unimp)

quit:
    return;

#undef DISPATCH
#undef EXEC
#undef TARGET
#else   // ENABLE_COMPUTED_GOTO = 0 // 這個版本比較慢，但好懂
    while (rv->csr_cycle < cycles_target && !rv->halt) {
        // fetch the next instruction
        inst = rv->io.mem_ifetch(rv, rv->PC);

        // standard uncompressed instruction
        if ((inst & 3) == 3) {
            index = (inst & INST_6_2) >> 2;

            // dispatch this opcode
            TABLE_TYPE op = jump_table[index];
            assert(op);
            if (!op(rv, inst))
                break;

            // increment the cycles csr
            rv->csr_cycle++;
        } else {
            // TODO: compressed instruction
            assert(!"Unreachable");
        }
    }
#endif  // ENABLE_COMPUTED_GOTO
}
```



以上的 goto 用到表格跳躍，參考下文範例

參考 -- https://stackoverflow.com/questions/938518/how-to-store-goto-labels-in-an-array-and-then-jump-to-them


```cpp
void *s[3] = {&&s0, &&s1, &&s2};

if (n >= 0 && n <=2)
    goto *s[n];

s0:
...
s1:
...
s2:
...
```

更完整範例，參考 : https://eli.thegreenplace.net/2012/07/12/computed-goto-for-efficient-dispatch-tables

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