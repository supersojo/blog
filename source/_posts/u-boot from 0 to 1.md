---
title: u-boot from 0 to 1
date: 2020-12-9 21:12:32
tags:
- uboot
- arm
- bootloader
categories:
- uboot
---


## u-boot from 0 to 1

这篇文章简单分析了u-boot从汇编代码到执行加载内核的整个过程，细节参考代码。

### u-boot.lds

```shell
su@ubuntu2004:/sdb1/u-boot-2020.10$ cat u-boot.lds
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
OUTPUT_ARCH(arm)
ENTRY(_start)
SECTIONS
{
 . = 0x00000000;
 . = ALIGN(4);
 .text :
 {
  *(.__image_copy_start)
  *(.vectors)
  arch/arm/cpu/armv7/start.o (.text*)
 }
 ...

```

u-boot链接脚本，u-boot入口为_start。

### arch/arm/lib/vectors.S

```assembly
/*
 * A macro to allow insertion of an ARM exception vector either
 * for the non-boot0 case or by a boot0-header.
 */
        .macro ARM_VECTORS
#ifdef CONFIG_ARCH_K3
        ldr     pc, _reset
#else
        b       reset
#endif
        ldr     pc, _undefined_instruction
        ldr     pc, _software_interrupt
        ldr     pc, _prefetch_abort
        ldr     pc, _data_abort
        ldr     pc, _not_used
        ldr     pc, _irq
        ldr     pc, _fiq
        .endm

.globl _start

/*
 *************************************************************************
 *
 * Vectors have their own section so linker script can map them easily
 *
 *************************************************************************
 */

        .section ".vectors", "ax"
_start:
        ARM_VECTORS
		...
_reset:                 .word reset
		...
```

中断向量表，如复位中断、为定义指令、数据中断、快中断、正常中断等等。

上电复位后执行复位中断。

### arch/arm/cpu/armv7/start.S

```assembly
/*************************************************************************
 *
 * Startup Code (reset vector)
 *
 * Do important init only if we don't start from memory!
 * Setup memory and board specific bits prior to relocation.
 * Relocate armboot to ram. Setup stack.
 *
 *************************************************************************/

        .globl  reset
        .globl  save_boot_params_ret
        .type   save_boot_params_ret,%function
#ifdef CONFIG_ARMV7_LPAE
        .global switch_to_hypervisor_ret
#endif

reset:
        /* Allow the board to save important registers */
        b       save_boot_params
save_boot_params_ret:
#ifdef CONFIG_ARMV7_LPAE
/*
 * check for Hypervisor support
 */
        mrc     p15, 0, r0, c0, c1, 1           @ read ID_PFR1
        and     r0, r0, #CPUID_ARM_VIRT_MASK    @ mask virtualization bits
        cmp     r0, #(1 << CPUID_ARM_VIRT_SHIFT)
        beq     switch_to_hypervisor
switch_to_hypervisor_ret:
#endif
        /*
         * disable interrupts (FIQ and IRQ), also set the cpu to SVC32 mode,
         * except if in HYP mode already
         */
        mrs     r0, cpsr
        and     r1, r0, #0x1f           @ mask mode bits
        teq     r1, #0x1a               @ test for HYP mode
        bicne   r0, r0, #0x1f           @ clear all mode bits
        orrne   r0, r0, #0x13           @ set SVC mode
        orr     r0, r0, #0xc0           @ disable FIQ and IRQ
        msr     cpsr,r0

/*
 * Setup vector:
 * (OMAP4 spl TEXT_BASE is not 32 byte aligned.
 * Continue to use ROM code vector only in OMAP4 spl)
 */
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
        /* Set V=0 in CP15 SCTLR register - for VBAR to point to vector */
        mrc     p15, 0, r0, c1, c0, 0   @ Read CP15 SCTLR Register
        bic     r0, #CR_V               @ V = 0
        mcr     p15, 0, r0, c1, c0, 0   @ Write CP15 SCTLR Register

#ifdef CONFIG_HAS_VBAR
        /* Set vector address in CP15 VBAR register */
        ldr     r0, =_start
        mcr     p15, 0, r0, c12, c0, 0  @Set VBAR
#endif
#endif

        /* the mask ROM code should have PLL and others stable */
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
#ifdef CONFIG_CPU_V7A
        bl      cpu_init_cp15
#endif
#ifndef CONFIG_SKIP_LOWLEVEL_INIT_ONLY
        bl      cpu_init_crit
#endif
#endif

        bl      _main

/*************************************************************************
 *
 * cpu_init_cp15
 *
 * Setup CP15 registers (cache, MMU, TLBs). The I-cache is turned on unless
 * CONFIG_SYS_ICACHE_OFF is defined.
 *
 *************************************************************************/
ENTRY(cpu_init_cp15)
        /*
         * Invalidate L1 I/D
         */
        mov     r0, #0                  @ set up for MCR
        mcr     p15, 0, r0, c8, c7, 0   @ invalidate TLBs
        mcr     p15, 0, r0, c7, c5, 0   @ invalidate icache
        mcr     p15, 0, r0, c7, c5, 6   @ invalidate BP array
        mcr     p15, 0, r0, c7, c10, 4  @ DSB
        mcr     p15, 0, r0, c7, c5, 4   @ ISB

        /*
         * disable MMU stuff and caches
         */
        mrc     p15, 0, r0, c1, c0, 0
        bic     r0, r0, #0x00002000     @ clear bits 13 (--V-)
        bic     r0, r0, #0x00000007     @ clear bits 2:0 (-CAM)
        orr     r0, r0, #0x00000002     @ set bit 1 (--A-) Align
        orr     r0, r0, #0x00000800     @ set bit 11 (Z---) BTB
#if CONFIG_IS_ENABLED(SYS_ICACHE_OFF)
        bic     r0, r0, #0x00001000     @ clear bit 12 (I) I-cache
#else
        orr     r0, r0, #0x00001000     @ set bit 12 (I) I-cache
#endif
        mcr     p15, 0, r0, c1, c0, 0

#ifdef CONFIG_ARM_ERRATA_716044
        mrc     p15, 0, r0, c1, c0, 0   @ read system control register
        orr     r0, r0, #1 << 11        @ set bit #11
        mcr     p15, 0, r0, c1, c0, 0   @ write system control register
#endif
		...
/* Early stack for ERRATA that needs into call C code */
#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
        ldr     r0, =(CONFIG_SPL_STACK)
#else
        ldr     r0, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
        bic     r0, r0, #7      /* 8-byte alignment for ABI compliance */
        mov     sp, r0
        ...
       mov     pc, r5                  @ back to my caller
ENDPROC(cpu_init_cp15)
		
```

从复位中断处理函数开始执行，首先关中断，禁止mmu，cache，准备c执行环境。

### arch/arm/lib/crt0.S

```assembly
/*
 * entry point of crt0 sequence
 */

ENTRY(_main)

/*
 * Set up initial C runtime environment and call board_init_f(0).
 */

#if defined(CONFIG_TPL_BUILD) && defined(CONFIG_TPL_NEEDS_SEPARATE_STACK)
        ldr     r0, =(CONFIG_TPL_STACK)
#elif defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_STACK)
        ldr     r0, =(CONFIG_SPL_STACK)
#else
        ldr     r0, =(CONFIG_SYS_INIT_SP_ADDR)
#endif
        bic     r0, r0, #7      /* 8-byte alignment for ABI compliance */
        mov     sp, r0
        bl      board_init_f_alloc_reserve
        mov     sp, r0
        /* set up gd here, outside any C code */
        mov     r9, r0
        bl      board_init_f_init_reserve

#if defined(CONFIG_SPL_BUILD) && defined(CONFIG_SPL_EARLY_BSS)
        CLEAR_BSS
#endif

        mov     r0, #0
        bl      board_init_f
#if ! defined(CONFIG_SPL_BUILD)

/*
 * Set up intermediate environment (new sp and gd) and call
 * relocate_code(addr_moni). Trick here is that we'll return
 * 'here' but relocated.
 */

        ldr     r0, [r9, #GD_START_ADDR_SP]     /* sp = gd->start_addr_sp */
        bic     r0, r0, #7      /* 8-byte alignment for ABI compliance */
        mov     sp, r0
        ldr     r9, [r9, #GD_NEW_GD]            /* r9 <- gd->new_gd */

        adr     lr, here
        ldr     r0, [r9, #GD_RELOC_OFF]         /* r0 = gd->reloc_off */
        add     lr, lr, r0
#if defined(CONFIG_CPU_V7M)
        orr     lr, #1                          /* As required by Thumb-only */
#endif
        ldr     r0, [r9, #GD_RELOCADDR]         /* r0 = gd->relocaddr */
        b       relocate_code
here:
/*
 * now relocate vectors
 */

        bl      relocate_vectors

/* Set up final (full) environment */

        bl      c_runtime_cpu_setup     /* we still call old routine here */
#endif
#if !defined(CONFIG_SPL_BUILD) || CONFIG_IS_ENABLED(FRAMEWORK)

#if !defined(CONFIG_SPL_BUILD) || !defined(CONFIG_SPL_EARLY_BSS)
        CLEAR_BSS
#endif

# ifdef CONFIG_SPL_BUILD
        /* Use a DRAM stack for the rest of SPL, if requested */
        bl      spl_relocate_stack_gd
        cmp     r0, #0
        movne   sp, r0
        movne   r9, r0
# endif

#if ! defined(CONFIG_SPL_BUILD)
        bl coloured_LED_init
        bl red_led_on
#endif
        /* call board_init_r(gd_t *id, ulong dest_addr) */
        mov     r0, r9                  /* gd_t */
        ldr     r1, [r9, #GD_RELOCADDR] /* dest_addr */
        /* call board_init_r */
#if CONFIG_IS_ENABLED(SYS_THUMB_BUILD)
        ldr     lr, =board_init_r       /* this is auto-relocated! */
        bx      lr
#else
        ldr     pc, =board_init_r       /./common/board_r.c* this is auto-relocated! */
#endif
        /* we should not return here. */

```

主要执行board_init_f函数， 重定位u-boot。

### common/board_r.c

```c
/*
 * We hope to remove most of the driver-related init and do it if/when
 * the driver is later used.
 *
 * TODO: perhaps reset the watchdog in the initcall function after each call?
 */
static init_fnc_t init_sequence_r[] = {
        initr_trace,
        initr_reloc,
		...
		run_main_loop,
};
static int run_main_loop(void)
{
#ifdef CONFIG_SANDBOX
        sandbox_main_loop_init();
#endif
        /* main_loop() can return to retry autoboot, if so just run it again */
        for (;;)
                main_loop();
        return 0;
}

void board_init_r(gd_t *new_gd, ulong dest_addr)
{
        /*
         * Set up the new global data pointer. So far only x86 does this
         * here.
         * TODO(sjg@chromium.org): Consider doing this for all archs, or
         * dropping the new_gd parameter.
         */
#if CONFIG_IS_ENABLED(X86_64)
        arch_setup_gd(new_gd);
#endif

#ifdef CONFIG_NEEDS_MANUAL_RELOC
        int i;
#endif

#if !defined(CONFIG_X86) && !defined(CONFIG_ARM) && !defined(CONFIG_ARM64)
        gd = new_gd;
#endif
        gd->flags &= ~GD_FLG_LOG_READY;

#ifdef CONFIG_NEEDS_MANUAL_RELOC
        for (i = 0; i < ARRAY_SIZE(init_sequence_r); i++)
                init_sequence_r[i] += gd->reloc_off;
#endif

        if (initcall_run_list(init_sequence_r))
                hang();

        /* NOTREACHED - run_main_loop() does not return */
        hang();
}

```

驱动等的初始化。

### common/main.c

```c
/* We come here after U-Boot is initialised and ready to process commands */
void main_loop(void)
{
        const char *s;

        bootstage_mark_name(BOOTSTAGE_ID_MAIN_LOOP, "main_loop");

        if (IS_ENABLED(CONFIG_VERSION_VARIABLE))
                env_set("ver", version_string);  /* set version variable */

        cli_init();

        if (IS_ENABLED(CONFIG_USE_PREBOOT))
                run_preboot_environment_command();

        if (IS_ENABLED(CONFIG_UPDATE_TFTP))
                update_tftp(0UL, NULL, NULL);

        s = bootdelay_process();
        if (cli_process_fdt(&s))
                cli_secure_boot_cmd(s);

        autoboot_command(s);

        cli_loop();
        panic("No CLI available");
}

```

进入main_loop，命令行初始化，准备执行u-boot命令。

### common/autoboot.c

```c
const char *bootdelay_process(void)
{
        char *s;
        int bootdelay;

        bootcount_inc();

        s = env_get("bootdelay");
        bootdelay = s ? (int)simple_strtol(s, NULL, 10) : CONFIG_BOOTDELAY;

        if (IS_ENABLED(CONFIG_OF_CONTROL))
                bootdelay = fdtdec_get_config_int(gd->fdt_blob, "bootdelay",
                                                  bootdelay);

        debug("### main_loop entered: bootdelay=%d\n\n", bootdelay);

        if (IS_ENABLED(CONFIG_AUTOBOOT_MENU_SHOW))
                bootdelay = menu_show(bootdelay);
        bootretry_init_cmd_timeout();

#ifdef CONFIG_POST
        if (gd->flags & GD_FLG_POSTFAIL) {
                s = env_get("failbootcmd");
        } else
#endif /* CONFIG_POST */
        if (bootcount_error())
                s = env_get("altbootcmd");
        else
                s = env_get("bootcmd");

        if (IS_ENABLED(CONFIG_OF_CONTROL))
                process_fdt_options(gd->fdt_blob);
        stored_bootdelay = bootdelay;
        
        return s;
}

void autoboot_command(const char *s)
{
        debug("### main_loop: bootcmd=\"%s\"\n", s ? s : "<UNDEFINED>");

        if (stored_bootdelay != -1 && s && !abortboot(stored_bootdelay)) {
                bool lock;
                int prev;

                lock = IS_ENABLED(CONFIG_AUTOBOOT_KEYED) &&
                        !IS_ENABLED(CONFIG_AUTOBOOT_KEYED_CTRLC);
                if (lock)
                        prev = disable_ctrlc(1); /* disable Ctrl-C checking */

                run_command_list(s, -1, 0);

                if (lock)
                        disable_ctrlc(prev);    /* restore Ctrl-C checking */
        }

        if (IS_ENABLED(CONFIG_USE_AUTOBOOT_MENUKEY) &&
            menukey == AUTOBOOT_MENUKEY) {
                s = env_get("menucmd");
                if (s)
                        run_command_list(s, -1, 0);
        }
}

```

获取bootcmd的设置，执行bootcmd的命令。

## References

1、u-boot源码





