# rv32emu-next

## install

```
guest@localhost:~/rv32emu-next$ su root
Password:
root@localhost:/home/guest/rv32emu-next# sudo apt install libsdl2-dev
Reading package lists... Done
Building dependency tree       
Reading state information... Done
libsdl2-dev is already the newest version (2.0.8+dfsg1-1ubuntu1.18.04.4).
0 upgraded, 0 newly installed, 0 to remove and 416 not upgraded.
```

## make & run

```
guest@localhost:~/rv32emu-next$ make
  CC    c_map.o
  CC    riscv.o
  CC    io.o
  CC    elf.o
  CC    main.o
  CC    syscall.o
  CC    syscall_sdl.o
  LD    build/rv32emu
guest@localhost:~/rv32emu-next$ make check
(cd build; ../build/rv32emu hello.elf)
Hello World!
inferior exit code 0
(cd build; ../build/rv32emu puzzle.elf)
success in 2005 trials
inferior exit code 0
```
