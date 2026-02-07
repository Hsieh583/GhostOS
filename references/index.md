# 📚 參考資料索引 (Reference Index)

本目錄收集 Intel 手冊和 xv6 源碼的重要片段分析。

---

## 1. 核心參考文件

### Intel 官方手冊

| 卷冊 | 名稱 | 重點內容 |
|------|------|----------|
| Volume 1 | 基本架構 | x86 概述、暫存器、資料類型 |
| Volume 2 | 指令集參考 | 所有 x86 指令詳解 |
| Volume 3A | 系統程式設計 (上) | 保護模式、分頁、中斷 |
| Volume 3B | 系統程式設計 (下) | APIC、虛擬化 |

### 重要章節

- **Volume 3A, Chapter 2**: System Architecture Overview
- **Volume 3A, Chapter 3**: Protected-Mode Memory Management
- **Volume 3A, Chapter 4**: Paging
- **Volume 3A, Chapter 6**: Interrupt and Exception Handling

---

## 2. xv6 源碼分析

xv6 是 MIT 開發的教學用作業系統，實現了 Unix V6 的核心功能。

### 目錄結構

```
xv6/
├── bootasm.S      # 啟動組語
├── bootmain.c     # 引導載入器
├── main.c         # 內核入口
├── proc.c         # 程序管理
├── vm.c           # 虛擬記憶體
├── trap.c         # 中斷處理
├── syscall.c      # 系統調用
├── fs.c           # 檔案系統
├── file.c         # 檔案操作
├── console.c      # 控制台
├── ide.c          # IDE 磁碟驅動
├── kbd.c          # 鍵盤驅動
└── ...
```

### 關鍵程式碼片段

#### bootasm.S - 進入保護模式

```asm
# 開啟 A20
seta20.1:
  inb     $0x64,%al
  testb   $0x2,%al
  jnz     seta20.1
  movb    $0xd1,%al
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al
  testb   $0x2,%al
  jnz     seta20.2
  movb    $0xdf,%al
  outb    %al,$0x60

# 載入 GDT
lgdt    gdtdesc

# 進入保護模式
movl    %cr0, %eax
orl     $CR0_PE, %eax
movl    %eax, %cr0
ljmp    $(SEG_KCODE<<3), $start32
```

#### vm.c - 頁表設置

```c
// 建立內核頁表
pde_t*
setupkvm(void)
{
  pde_t *pgdir;
  struct kmap *k;

  if((pgdir = (pde_t*)kalloc()) == 0)
    return 0;
  memset(pgdir, 0, PGSIZE);
  
  for(k = kmap; k < &kmap[NELEM(kmap)]; k++)
    if(mappages(pgdir, k->virt, k->phys_end - k->phys_start,
                (uint)k->phys_start, k->perm) < 0) {
      freevm(pgdir);
      return 0;
    }
  return pgdir;
}
```

#### trap.c - 中斷處理

```c
void
trap(struct trapframe *tf)
{
  if(tf->trapno == T_SYSCALL){
    if(myproc()->killed)
      exit();
    myproc()->tf = tf;
    syscall();
    if(myproc()->killed)
      exit();
    return;
  }

  switch(tf->trapno){
  case T_IRQ0 + IRQ_TIMER:
    // 計時器中斷
    acquire(&tickslock);
    ticks++;
    wakeup(&ticks);
    release(&tickslock);
    lapiceoi();
    break;
  // ...
  }
}
```

---

## 3. OSDev Wiki 重要條目

| 主題 | URL |
|------|-----|
| 引導載入器 | https://wiki.osdev.org/Bootloader |
| 保護模式 | https://wiki.osdev.org/Protected_Mode |
| 分頁 | https://wiki.osdev.org/Paging |
| GDT | https://wiki.osdev.org/Global_Descriptor_Table |
| IDT | https://wiki.osdev.org/Interrupt_Descriptor_Table |
| 系統調用 | https://wiki.osdev.org/System_Calls |
| PS/2 鍵盤 | https://wiki.osdev.org/PS/2_Keyboard |
| VGA | https://wiki.osdev.org/VGA_Hardware |
| ATA | https://wiki.osdev.org/ATA_PIO_Mode |

---

## 4. 推薦書籍

| 書名 | 作者 | 重點 |
|------|------|------|
| Operating Systems: Three Easy Pieces | Arpaci-Dusseau | 作業系統概念 |
| Operating System Concepts | Silberschatz | 經典教科書 |
| Modern Operating Systems | Tanenbaum | 深入分析 |
| Linux Kernel Development | Robert Love | Linux 內核 |
| Understanding the Linux Kernel | Bovet & Cesati | Linux 詳解 |

---

## 5. 線上資源

### 教學系列

- [OSDev Bare Bones Tutorial](https://wiki.osdev.org/Bare_Bones)
- [JamesM's Kernel Development](http://www.jamesmolloy.co.uk/tutorial_html/)
- [BrokenThorn Entertainment OS Development](http://www.brokenthorn.com/Resources/OSDevIndex.html)

### 開源作業系統

| 專案 | 描述 | 難度 |
|------|------|------|
| xv6 | MIT 教學 OS | ⭐⭐ |
| ToaruOS | 功能完整的 hobby OS | ⭐⭐⭐ |
| SerenityOS | 現代 Unix-like OS | ⭐⭐⭐⭐ |
| Minix | 微內核架構 | ⭐⭐⭐ |

---

## 6. 調試工具

### QEMU

```bash
# 啟動 QEMU 並等待 GDB 連接
qemu-system-i386 -kernel kernel.bin -s -S

# GDB 連接
gdb kernel.bin
(gdb) target remote localhost:1234
(gdb) break kernel_main
(gdb) continue
```

### Bochs

```bash
# bochs 配置文件範例
megs: 32
romimage: file=/usr/share/bochs/BIOS-bochs-latest
vgaromimage: file=/usr/share/bochs/VGABIOS-lgpl-latest
boot: disk
ata0-master: type=disk, path="disk.img", mode=flat
log: bochs.log
magic_break: enabled=1
```

### objdump

```bash
# 反組譯內核
objdump -d kernel.bin > kernel.asm

# 查看段資訊
objdump -h kernel.bin

# 查看符號表
objdump -t kernel.bin
```

---

## 後續學習路徑

1. **基礎階段**
   - 理解 CPU 啟動流程
   - 實作簡單的引導程序
   - 進入保護模式

2. **中級階段**
   - 設置分頁
   - 實作中斷處理
   - 基本鍵盤和螢幕驅動

3. **進階階段**
   - 程序調度
   - 虛擬記憶體管理
   - 檔案系統

4. **專家階段**
   - 多核心支援
   - 網路協定棧
   - 圖形介面
