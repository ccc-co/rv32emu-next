# SDL

https://www.itread01.com/content/1549641630.html

摘要：

關於SDL,在簡介裡面，有一些概念，但是很多人還是留言，不清楚到底是個什麼。這節，我簡單總結下:

我們如何將一張圖顯示在螢幕上。這裡簡單的分為幾個部分，硬體螢幕，驅動程式，軟體部分。SDL不直接關注硬體螢幕，而是關注每個平臺下的螢幕驅動程式。比如window下的DirectX,linux下的x11 ,以及android下的opengl es。SDL通過將這三個平臺，當然不止這三個平臺的螢幕驅動，封裝成一套對外統一的API呼叫，讓使用者可以不關注具體某個平臺，可以快速開發影象的繪製操作。SDL的核心，便是如此。

記住，sdl的講解，可以作為興趣學習，它不是成熟的遊戲開發引擎，但卻是非常成熟，跨平臺的一套渲染引擎。它本身做的事情非常簡單，就是讓一張圖片，可以顯示到更多平臺，同時附加一些音訊編解碼而已。誠然，sdl不是你開發遊戲的首選，但卻是不可多得的，程式碼不算複雜，可以深入學習，掌握一套跨平臺的開發流程，思路，同時理解影象格式的分類，轉化，運算的具體實現。

## rv32emu-next 中的 SDL

syscall.c 當中有下列段落

```cpp

#ifdef ENABLE_SDL
extern void syscall_draw_frame(struct riscv_t *rv);
extern void syscall_draw_frame_pal(struct riscv_t *rv);
#endif

void syscall_handler(struct riscv_t *rv)
{
    /* get the syscall number */
    riscv_word_t syscall = rv_get_reg(rv, rv_reg_a7);

    switch (syscall) { /* dispatch system call */
    case SYS_close:
        syscall_close(rv);
        break;
    case SYS_lseek:
        syscall_lseek(rv);
        break;
    case SYS_read:
        syscall_read(rv);
        break;
    case SYS_write:
        syscall_write(rv);
        break;
    case SYS_fstat:
        syscall_fstat(rv);
        break;
    case SYS_brk:
        syscall_brk(rv);
        break;
    case SYS_exit:
        syscall_exit(rv);
        break;
    case SYS_gettimeofday:
        syscall_gettimeofday(rv);
        break;
    case SYS_open:
        syscall_open(rv);
        break;
#ifdef ENABLE_SDL
    case 0xbeef:
        syscall_draw_frame(rv);
        break;
    case 0xbabe:
        syscall_draw_frame_pal(rv);
        break;
#endif
    default:
        fprintf(stderr, "unknown syscall %d\n", (int) syscall);
        rv_halt(rv);
        break;
    }
}

```

利用自行增加的系統呼叫，透過 SDL 達到畫面更新的效果。
