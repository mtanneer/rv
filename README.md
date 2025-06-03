# rv

RISC-V CPU core written in ANSI C.

Features:

- `RV32IMAC_Zicsr` implementation with M-mode and S-mode
- Boots RISCV32 Linux
- Passes all supported tests in [`riscv-tests`](https://github.com/riscv/riscv-tests)
- ~800 lines of code
- Doesn't use any integer types larger than 32 bits, even for multiplication
- Simple API (two required functions, plus one memory callback function that you provide)
- No memory allocations

## API

```c
/* Memory access callback: data is input/output, return RV_BAD on fault. */
typedef rv_res (*rv_bus_cb)(void *user, rv_u32 addr, rv_u8 *data, rv_u32 is_store, rv_u32 width);

/* Initialize CPU. You can call this again on `cpu` to reset it. */
void rv_init(rv *cpu, void *user, rv_bus_cb bus_cb);

/* Single-step CPU. Returns RV_E* on exception. */
rv_u32 rv_step(rv *cpu);
```

## Usage

```c
#include <stdio.h>
#include <string.h>

#include "rv.h"

#define RAM_BASE 0x80000000
#define RAM_SIZE 0x10000

rv_res bus_cb(void *user, rv_u32 addr, rv_u8 *data, rv_u32 is_store,
              rv_u32 width) {
  rv_u8 *mem = (rv_u8 *)user + addr - RAM_BASE;
  if (addr < RAM_BASE || addr + width >= RAM_BASE + RAM_SIZE)
    return RV_BAD;
  memcpy(is_store ? mem : data, is_store ? data : mem, width);
  return RV_OK;
}

rv_u32 program[2] = {
    /*            */             /* _start: */
    /* 0x80000000 */ 0x02A88893, /* add a7, a7, 42 */
    /* 0x80000004 */ 0x00000073  /* ecall */
};

int main(void) {
  rv_u8 mem[RAM_SIZE];
  rv cpu;
  rv_init(&cpu, (void *)mem, &bus_cb);
  memcpy((void *)mem, (void *)program, sizeof(program));
  while (rv_step(&cpu) != RV_EMECALL) {
  }
  printf("Environment call @ %08X: %u\n", cpu.csr.mepc, cpu.r[17]);
  return 0;
}
```

See [`tools/example/example.c`](tools/example/example.c).

## Running Linux

This repository contains a machine emulator that can use `rv` to boot Linux.
See [`tools/linux/README.md`](tools/linux/README.md).

## Targeting `rv`

Use [riscv-gnu-toolchain](https://github.com/riscv-collab/riscv-gnu-toolchain) with [tools/link.ld](tools/link.ld).

Suggested GCC commandline:

`riscv64-unknown-elf-gcc example.S -nostdlib -nostartfiles -Tlink.ld -march=rv32imac -mabi=ilp32 -o example.o -e _start -g -no-pie`

To dump a binary starting at `0x80000000` that can be directly loaded by `rv` as in the above example:

`riscv64-unknown-elf-objcopy -g -O binary example.o example.bin`

## Instruction List

Click an instruction to see its implementation in `rv.c`.

- [`add       `](rv.c#L560)[`addi      `](rv.c#L560)[`amoadd.w  `](rv.c#L519)[`amoand.w  `](rv.c#L531)[`amomax.w  `](rv.c#L535)[`amomaxu.w `](rv.c#L539)[`amomin.w  `](rv.c#L533)[`amominu.w `](rv.c#L537)
- [`amoor.w   `](rv.c#L529)[`amoswap.w `](rv.c#L521)[`amoxor.w  `](rv.c#L527)[`and       `](rv.c#L577)[`andi      `](rv.c#L577)[`auipc     `](rv.c#L669)[`beq       `](rv.c#L480)[`bge       `](rv.c#L483)
- [`bgtu      `](rv.c#L485)[`blt       `](rv.c#L482)[`bltu      `](rv.c#L484)[`bne       `](rv.c#L481)[`c.add     `](rv.c#L357)[`c.addi    `](rv.c#L295)[`c.addi16sp`](rv.c#L302)[`c.and     `](rv.c#L323)
- [`c.andi    `](rv.c#L314)[`c.beqz    `](rv.c#L333)[`c.bnez    `](rv.c#L335)[`c.ebreak  `](rv.c#L354)[`c.j       `](rv.c#L331)[`c.jal     `](rv.c#L297)[`c.jalr    `](rv.c#L351)[`c.jr      `](rv.c#L346)
- [`c.li      `](rv.c#L299)[`c.lui     `](rv.c#L304)[`c.lw      `](rv.c#L287)[`c.lwsp    `](rv.c#L343)[`c.mv      `](rv.c#L348)[`c.or      `](rv.c#L321)[`c.slli    `](rv.c#L341)[`c.srai    `](rv.c#L312)
- [`c.srli    `](rv.c#L310)[`c.sub     `](rv.c#L317)[`c.sw      `](rv.c#L289)[`c.swsp    `](rv.c#L359)[`c.xor     `](rv.c#L319)[`csrrc     `](rv.c#L622)[`csrrci    `](rv.c#L622)[`csrrs     `](rv.c#L616)
- [`csrrsi    `](rv.c#L616)[`csrrw     `](rv.c#L607)[`csrrwi    `](rv.c#L607)[`div       `](rv.c#L591)[`divu      `](rv.c#L593)[`ebreak    `](rv.c#L658)[`ecall     `](rv.c#L655)[`fence     `](rv.c#L502)
- [`fence.i   `](rv.c#L506)[`jal       `](rv.c#L548)[`jalr      `](rv.c#L495)[`lb        `](rv.c#L459)[`lbu       `](rv.c#L459)[`lh        `](rv.c#L459)[`lhu       `](rv.c#L459)[`lr.w      `](rv.c#L523)
- [`lui       `](rv.c#L671)[`lw        `](rv.c#L459)[`mret      `](rv.c#L631)[`mul       `](rv.c#L581)[`mulh      `](rv.c#L581)[`mulhsu    `](rv.c#L581)[`mulhu     `](rv.c#L581)[`or        `](rv.c#L575)
- [`ori       `](rv.c#L575)[`rem       `](rv.c#L595)[`remu      `](rv.c#L597)[`sb        `](rv.c#L471)[`sc.w      `](rv.c#L525)[`sfence.vma`](rv.c#L651)[`sh        `](rv.c#L471)[`sll       `](rv.c#L565)
- [`slli      `](rv.c#L565)[`slt       `](rv.c#L567)[`slti      `](rv.c#L567)[`sltiu     `](rv.c#L569)[`sltu      `](rv.c#L569)[`sra       `](rv.c#L573)[`srai      `](rv.c#L573)[`sret      `](rv.c#L631)
- [`srl       `](rv.c#L573)[`srli      `](rv.c#L573)[`sub       `](rv.c#L560)[`sw        `](rv.c#L471)[`wfi       `](rv.c#L648)[`xor       `](rv.c#L571)[`xori      `](rv.c#L571)

## FAQ

### Spaghetti code!

- `rv` was written in a way that takes maximal advantage of RISCV's instruction orthogonality.
- `rv` also tries to strike a good balance between conciseness and readability.
- Of course, being able to read this code at all requires intimate prior knowledge of the ISA encoding.

### No switch statements!

- C only allows constant expressions in switch statements. In addition to an abundance of `break` statements using these would result in more bloated code in the author's opinion. As it turns out, you are actually free to reimplement this code with switch statements. See [LICENSE.txt](LICENSE.txt).

### Not useful!

- [Ok](https://www.google.com/search?q=happy+smiley+thumbs+up+happy+cool+funny+ok&tbm=isch)

### Slow!

- [Ok](https://www.google.com/search?q=happy+smiley+thumbs+up+happy+cool+funny+ok&tbm=isch)

## Caveats

- Written in C89.
- Not actually written in C89, since it uses external names longer than 6 characters.
- Doesn't use any integer types larger than 32 bits, even for multiplication, because it's written in C89.
- Assumes width of integer types in a way that's not completely compliant with C89/99. Fix for this is coming soon, I'm working on a watertight `<stdint.h>` for C89.
- Written in C89.
