# 📝 xv6 源碼分析筆記

## 1. 啟動流程分析

### 啟動順序

```
BIOS
  │
  ▼
bootasm.S (16-bit real mode)
  │ 開啟 A20
  │ 載入 GDT
  │ 進入 protected mode
  ▼
bootmain.c (32-bit protected mode)
  │ 從磁碟載入 kernel ELF
  │ 跳轉到 kernel entry
  ▼
entry.S
  │ 設置簡單頁表
  │ 啟用分頁
  ▼
main.c: main()
  │ 初始化各子系統
  │ 啟動第一個程序
  ▼
多工執行
```

---

## 2. 記憶體佈局

### 虛擬位址空間 (xv6)

```
0x80000000 ┌─────────────────────────────┐ KERNBASE
           │                             │
           │      Kernel Space           │
           │                             │
           │      直接映射到              │
           │      0x00000000 開始的       │
           │      物理記憶體              │
           │                             │
0xFE000000 ├─────────────────────────────┤ DEVSPACE
           │      Memory-mapped I/O      │
0xFFFFFFFF └─────────────────────────────┘

0x00000000 ┌─────────────────────────────┐
           │      Text & Data            │
           ├─────────────────────────────┤
           │      Stack                  │
           │      (1 page, guard page    │
           │       below)                │
           ├─────────────────────────────┤
           │      Heap                   │
           │      (grows up)             │
           │                             │
0x80000000 └─────────────────────────────┘ KERNBASE
              User Space
```

### 關鍵常數 (memlayout.h)

```c
#define EXTMEM  0x100000            // 擴展記憶體起始
#define PHYSTOP 0xE000000           // 物理記憶體頂端 (224 MB)
#define DEVSPACE 0xFE000000         // 設備空間起始

#define KERNBASE 0x80000000         // 內核虛擬位址起始
#define KERNLINK (KERNBASE+EXTMEM)  // 內核 link 位址

#define V2P(a) (((uint)(a)) - KERNBASE)
#define P2V(a) ((void *)(((char *)(a)) + KERNBASE))

#define V2P_WO(x) ((x) - KERNBASE)  // without casts
#define P2V_WO(x) ((x) + KERNBASE)
```

---

## 3. 程序結構

### struct proc (proc.h)

```c
struct proc {
  uint sz;                     // 程序記憶體大小 (bytes)
  pde_t* pgdir;                // 頁目錄
  char *kstack;                // 內核堆疊
  enum procstate state;        // 程序狀態
  int pid;                     // 程序 ID
  struct proc *parent;         // 父程序
  struct trapframe *tf;        // Trap frame
  struct context *context;     // 上下文 (用於 switch)
  void *chan;                  // 如果非零，正在等待
  int killed;                  // 如果非零，已被殺死
  struct file *ofile[NOFILE];  // 開啟的檔案
  struct inode *cwd;           // 當前目錄
  char name[16];               // 程序名稱
};
```

### 程序狀態

```c
enum procstate {
  UNUSED,     // 未使用
  EMBRYO,     // 正在建立
  SLEEPING,   // 睡眠中
  RUNNABLE,   // 可執行
  RUNNING,    // 執行中
  ZOMBIE      // 殭屍 (等待 wait())
};
```

---

## 4. 上下文切換

### struct context (proc.h)

```c
struct context {
  uint edi;
  uint esi;
  uint ebx;
  uint ebp;
  uint eip;
};
```

### swtch.S

```asm
# void swtch(struct context **old, struct context *new);

.globl swtch
swtch:
  movl 4(%esp), %eax     # old
  movl 8(%esp), %edx     # new

  # 保存舊的 callee-saved 暫存器
  pushl %ebp
  pushl %ebx
  pushl %esi
  pushl %edi

  # 切換堆疊
  movl %esp, (%eax)      # 保存舊的 esp
  movl %edx, %esp        # 載入新的 esp

  # 載入新的 callee-saved 暫存器
  popl %edi
  popl %esi
  popl %ebx
  popl %ebp
  ret
```

### scheduler (proc.c)

```c
void
scheduler(void)
{
  struct proc *p;
  struct cpu *c = mycpu();
  c->proc = 0;
  
  for(;;){
    // 啟用中斷
    sti();

    // 遍歷程序表找可執行的程序
    acquire(&ptable.lock);
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      // 切換到這個程序
      c->proc = p;
      switchuvm(p);
      p->state = RUNNING;

      swtch(&(c->scheduler), p->context);
      switchkvm();

      // 程序已經完成了這輪執行
      c->proc = 0;
    }
    release(&ptable.lock);
  }
}
```

---

## 5. 系統調用

### 系統調用路徑

```
User space:          write(fd, buf, n)
                           │
                           ▼
usys.S:              .globl write
                     write:
                       movl $SYS_write, %eax
                       int $T_SYSCALL
                       ret
                           │
                           ▼
vectors.S:           vector64:  # T_SYSCALL = 64
                       pushl $0
                       pushl $64
                       jmp alltraps
                           │
                           ▼
trapasm.S:           alltraps:
                       # 建立 trap frame
                       pushal
                       pushl %ds
                       pushl %es
                       pushl %fs
                       pushl %gs
                       movw $(SEG_KDATA<<3), %ax
                       movw %ax, %ds
                       movw %ax, %es
                       pushl %esp  # trapframe 指標
                       call trap
                           │
                           ▼
trap.c:              trap(struct trapframe *tf)
                       if(tf->trapno == T_SYSCALL) {
                         myproc()->tf = tf;
                         syscall();
                         return;
                       }
                           │
                           ▼
syscall.c:           syscall()
                       int num = curproc->tf->eax;
                       curproc->tf->eax = 
                         syscalls[num]();
                           │
                           ▼
sysfile.c:           sys_write()
                       # 實際處理
```

### syscall 表 (syscall.c)

```c
static int (*syscalls[])(void) = {
[SYS_fork]    sys_fork,
[SYS_exit]    sys_exit,
[SYS_wait]    sys_wait,
[SYS_pipe]    sys_pipe,
[SYS_read]    sys_read,
[SYS_kill]    sys_kill,
[SYS_exec]    sys_exec,
[SYS_fstat]   sys_fstat,
[SYS_chdir]   sys_chdir,
[SYS_dup]     sys_dup,
[SYS_getpid]  sys_getpid,
[SYS_sbrk]    sys_sbrk,
[SYS_sleep]   sys_sleep,
[SYS_uptime]  sys_uptime,
[SYS_open]    sys_open,
[SYS_write]   sys_write,
[SYS_mknod]   sys_mknod,
[SYS_unlink]  sys_unlink,
[SYS_link]    sys_link,
[SYS_mkdir]   sys_mkdir,
[SYS_close]   sys_close,
};
```

---

## 6. 檔案系統

### 磁碟佈局

```
┌─────────────────────────────────────────────┐
│  Block 0:  (unused, boot block)             │
├─────────────────────────────────────────────┤
│  Block 1:  super block                      │
├─────────────────────────────────────────────┤
│  Block 2 - ?:  log                          │
├─────────────────────────────────────────────┤
│  Block ?:  inode blocks                     │
├─────────────────────────────────────────────┤
│  Block ?:  bitmap block                     │
├─────────────────────────────────────────────┤
│  Block ? - end:  data blocks                │
└─────────────────────────────────────────────┘
```

### inode 結構 (fs.h)

```c
// 磁碟上的 inode
struct dinode {
  short type;           // 檔案類型
  short major;          // 主設備號 (T_DEV only)
  short minor;          // 次設備號 (T_DEV only)
  short nlink;          // 連結數
  uint size;            // 檔案大小 (bytes)
  uint addrs[NDIRECT+1]; // 資料區塊位址
};

// 記憶體中的 inode
struct inode {
  uint dev;           // 設備號
  uint inum;          // Inode 號
  int ref;            // 參考計數
  struct sleeplock lock;
  int valid;          // inode 是否從磁碟讀取過

  short type;
  short major;
  short minor;
  short nlink;
  uint size;
  uint addrs[NDIRECT+1];
};
```

---

## 7. 鎖機制

### spinlock (spinlock.h)

```c
struct spinlock {
  uint locked;       // 是否被持有

  // 調試資訊:
  char *name;        // 鎖名稱
  struct cpu *cpu;   // 持有鎖的 CPU
  uint pcs[10];      // 呼叫堆疊
};
```

### acquire/release (spinlock.c)

```c
void
acquire(struct spinlock *lk)
{
  pushcli();  // 禁止中斷
  if(holding(lk))
    panic("acquire");

  // 自旋直到獲得鎖
  while(xchg(&lk->locked, 1) != 0)
    ;

  // 記錄調試資訊
  __sync_synchronize();
  lk->cpu = mycpu();
  getcallerpcs(&lk, lk->pcs);
}

void
release(struct spinlock *lk)
{
  if(!holding(lk))
    panic("release");

  lk->pcs[0] = 0;
  lk->cpu = 0;

  __sync_synchronize();
  
  // 釋放鎖
  asm volatile("movl $0, %0" : "+m" (lk->locked) : );

  popcli();  // 恢復中斷
}
```

---

## 8. 學習要點總結

1. **模組化設計**：xv6 將各子系統分離得很清楚
2. **最小化實現**：專注於核心概念，省略複雜功能
3. **清晰的抽象層**：VFS、設備驅動介面設計良好
4. **完整的教學價值**：包含啟動、記憶體、程序、檔案系統所有核心概念

### 推薦閱讀順序

1. bootasm.S, bootmain.c - 理解啟動
2. entry.S, main.c - 理解內核初始化
3. vm.c - 理解虛擬記憶體
4. proc.c - 理解程序管理
5. trap.c, syscall.c - 理解中斷和系統調用
6. fs.c - 理解檔案系統
