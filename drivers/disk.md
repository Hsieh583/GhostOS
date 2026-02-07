# 💾 磁碟驅動抽象層 (Disk Driver)

## 概述

磁碟驅動負責與儲存設備通訊。本文檔分析 ATA/IDE 硬碟的工作原理。

---

## 1. ATA 硬體介面

### I/O 端口 (Primary Channel)

| 端口 | 讀取用途 | 寫入用途 |
|------|----------|----------|
| 0x1F0 | 資料暫存器 | 資料暫存器 |
| 0x1F1 | 錯誤暫存器 | 特性暫存器 |
| 0x1F2 | 扇區計數 | 扇區計數 |
| 0x1F3 | LBA 低位元組 | LBA 低位元組 |
| 0x1F4 | LBA 中位元組 | LBA 中位元組 |
| 0x1F5 | LBA 高位元組 | LBA 高位元組 |
| 0x1F6 | 驅動/標頭 | 驅動/標頭 |
| 0x1F7 | 狀態暫存器 | 命令暫存器 |
| 0x3F6 | 替代狀態 | 設備控制 |

### 狀態暫存器

```
位元 7 (BSY):  忙碌中
位元 6 (DRDY): 驅動就緒
位元 5 (DF):   驅動故障
位元 4 (SRV):  服務請求
位元 3 (DRQ):  資料請求就緒
位元 2 (CORR): 已修正錯誤
位元 1 (IDX):  索引標記
位元 0 (ERR):  錯誤
```

---

## 2. PIO 模式讀取

### 流程圖

```
┌─────────────────────────────────────────┐
│           開始讀取扇區                   │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    等待 BSY = 0 (驅動不忙碌)             │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    設置驅動/標頭暫存器 (0x1F6)           │
│    • 位元 7-5: 101 (LBA 模式)           │
│    • 位元 4: 驅動選擇 (0=主, 1=從)      │
│    • 位元 3-0: LBA 最高 4 位元          │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    設置扇區計數 (0x1F2)                  │
│    設置 LBA 位址 (0x1F3-0x1F5)          │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    發送讀取命令 0x20 到 0x1F7            │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    等待 BSY = 0 且 DRQ = 1              │
│    (資料準備好)                          │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    從 0x1F0 讀取 256 個 16-bit 字       │
│    (總共 512 bytes = 1 扇區)            │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│    如果還有更多扇區，重複上述步驟        │
└─────────────────────────────────────────┘
```

### 程式碼範例

```c
// ATA 命令
#define ATA_CMD_READ_PIO         0x20
#define ATA_CMD_READ_PIO_EXT     0x24
#define ATA_CMD_WRITE_PIO        0x30
#define ATA_CMD_WRITE_PIO_EXT    0x34
#define ATA_CMD_IDENTIFY         0xEC

// 讀取扇區
void ata_read_sectors(uint8_t drive, uint32_t lba, 
                      uint8_t sectors, void* buffer) {
    // 等待驅動就緒
    ata_wait_ready();
    
    // 選擇驅動和設置 LBA
    outb(0x1F6, 0xE0 | (drive << 4) | ((lba >> 24) & 0x0F));
    
    // 設置扇區計數
    outb(0x1F2, sectors);
    
    // 設置 LBA
    outb(0x1F3, (uint8_t)(lba));
    outb(0x1F4, (uint8_t)(lba >> 8));
    outb(0x1F5, (uint8_t)(lba >> 16));
    
    // 發送讀取命令
    outb(0x1F7, ATA_CMD_READ_PIO);
    
    // 讀取資料
    uint16_t* ptr = (uint16_t*)buffer;
    for (int i = 0; i < sectors; i++) {
        // 等待資料就緒
        ata_wait_drq();
        
        // 讀取 256 個 16-bit 字
        for (int j = 0; j < 256; j++) {
            *ptr++ = inw(0x1F0);
        }
    }
}

// 等待函數
static void ata_wait_ready(void) {
    // 讀取替代狀態以清除中斷
    inb(0x3F6);
    inb(0x3F6);
    inb(0x3F6);
    inb(0x3F6);
    
    // 等待 BSY 清除
    while (inb(0x1F7) & 0x80);
}

static void ata_wait_drq(void) {
    // 等待 BSY 清除且 DRQ 設置
    uint8_t status;
    do {
        status = inb(0x1F7);
    } while ((status & 0x80) || !(status & 0x08));
}
```

---

## 3. IDENTIFY 命令

### 獲取驅動資訊

```c
typedef struct {
    uint16_t general_config;          // 0
    uint16_t cylinders;               // 1
    uint16_t reserved1;               // 2
    uint16_t heads;                   // 3
    uint16_t reserved2[2];            // 4-5
    uint16_t sectors_per_track;       // 6
    uint16_t reserved3[3];            // 7-9
    char serial_number[20];           // 10-19
    uint16_t reserved4[3];            // 20-22
    char firmware_revision[8];        // 23-26
    char model_number[40];            // 27-46
    uint16_t max_sectors_per_int;     // 47
    uint16_t reserved5;               // 48
    uint16_t capabilities[2];         // 49-50
    uint16_t reserved6[2];            // 51-52
    uint16_t field_validity;          // 53
    uint16_t reserved7[5];            // 54-58
    uint16_t sectors_per_int;         // 59
    uint32_t total_sectors;           // 60-61 (LBA28)
    uint16_t reserved8[2];            // 62-63
    uint16_t multiword_dma_modes;     // 64
    uint16_t pio_modes;               // 65
    // ... 更多
    uint64_t total_sectors_lba48;     // 100-103 (LBA48)
} __attribute__((packed)) ata_identify_t;

bool ata_identify(uint8_t drive, ata_identify_t* info) {
    // 選擇驅動
    outb(0x1F6, 0xA0 | (drive << 4));
    
    // 清除其他暫存器
    outb(0x1F2, 0);
    outb(0x1F3, 0);
    outb(0x1F4, 0);
    outb(0x1F5, 0);
    
    // 發送 IDENTIFY 命令
    outb(0x1F7, ATA_CMD_IDENTIFY);
    
    // 檢查驅動是否存在
    uint8_t status = inb(0x1F7);
    if (status == 0) {
        return false;  // 驅動不存在
    }
    
    // 等待 BSY 清除
    while (inb(0x1F7) & 0x80);
    
    // 檢查 LBA 暫存器 (應該為 0 for ATA)
    if (inb(0x1F4) != 0 || inb(0x1F5) != 0) {
        return false;  // 不是 ATA 驅動 (可能是 ATAPI)
    }
    
    // 等待 DRQ 或 ERR
    do {
        status = inb(0x1F7);
    } while (!(status & 0x08) && !(status & 0x01));
    
    if (status & 0x01) {
        return false;  // 錯誤
    }
    
    // 讀取 IDENTIFY 資料
    uint16_t* ptr = (uint16_t*)info;
    for (int i = 0; i < 256; i++) {
        *ptr++ = inw(0x1F0);
    }
    
    // 轉換字串 (ATA 使用 big-endian 字串)
    swap_bytes(info->serial_number, 20);
    swap_bytes(info->firmware_revision, 8);
    swap_bytes(info->model_number, 40);
    
    return true;
}
```

---

## 4. LBA 定址模式

### CHS vs LBA

```
CHS (Cylinder-Head-Sector):
┌─────────────────────────────────────────┐
│  傳統定址方式                            │
│  • Cylinder: 0-65535                    │
│  • Head: 0-15                           │
│  • Sector: 1-63                         │
│  • 最大容量: 約 8 GB                     │
└─────────────────────────────────────────┘

LBA28 (Logical Block Addressing):
┌─────────────────────────────────────────┐
│  線性定址方式                            │
│  • 28-bit 位址                          │
│  • 最大容量: 128 GB                      │
└─────────────────────────────────────────┘

LBA48:
┌─────────────────────────────────────────┐
│  擴展線性定址                            │
│  • 48-bit 位址                          │
│  • 最大容量: 128 PB                      │
└─────────────────────────────────────────┘
```

### LBA 到 CHS 轉換

```c
void lba_to_chs(uint32_t lba, uint16_t* cylinder, 
                uint8_t* head, uint8_t* sector) {
    // 假設: 每軌 63 扇區, 每柱面 16 磁頭
    const uint8_t sectors_per_track = 63;
    const uint8_t heads_per_cylinder = 16;
    
    *sector = (lba % sectors_per_track) + 1;
    *head = (lba / sectors_per_track) % heads_per_cylinder;
    *cylinder = lba / (sectors_per_track * heads_per_cylinder);
}
```

---

## 5. 分區表

### MBR 分區表結構

```
偏移量 0x1BE (446 bytes):
┌────────────────────────────────────────┐
│            分區條目 1 (16 bytes)        │
├────────────────────────────────────────┤
│            分區條目 2 (16 bytes)        │
├────────────────────────────────────────┤
│            分區條目 3 (16 bytes)        │
├────────────────────────────────────────┤
│            分區條目 4 (16 bytes)        │
└────────────────────────────────────────┘
```

### 分區條目格式

| 偏移 | 大小 | 說明 |
|------|------|------|
| 0 | 1 | 啟動標誌 (0x80 = 可啟動) |
| 1-3 | 3 | 起始 CHS |
| 4 | 1 | 分區類型 |
| 5-7 | 3 | 結束 CHS |
| 8-11 | 4 | 起始 LBA |
| 12-15 | 4 | 扇區總數 |

### 常見分區類型

| 類型 | 描述 |
|------|------|
| 0x00 | 空 |
| 0x01 | FAT12 |
| 0x04 | FAT16 (<32MB) |
| 0x06 | FAT16 |
| 0x07 | NTFS |
| 0x0B | FAT32 (CHS) |
| 0x0C | FAT32 (LBA) |
| 0x82 | Linux Swap |
| 0x83 | Linux Native |
| 0xEE | GPT 保護 MBR |

---

## 6. 錯誤處理

```c
// 錯誤暫存器位元
#define ATA_ERR_AMNF   0x01  // 找不到位址標記
#define ATA_ERR_TK0NF  0x02  // 找不到軌道 0
#define ATA_ERR_ABRT   0x04  // 命令中止
#define ATA_ERR_MCR    0x08  // 媒體變更請求
#define ATA_ERR_IDNF   0x10  // 找不到 ID
#define ATA_ERR_MC     0x20  // 媒體已變更
#define ATA_ERR_UNC    0x40  // 不可修正的資料錯誤
#define ATA_ERR_BBK    0x80  // 壞區塊

const char* ata_get_error_string(uint8_t error) {
    if (error & ATA_ERR_BBK)  return "Bad Block";
    if (error & ATA_ERR_UNC)  return "Uncorrectable Data Error";
    if (error & ATA_ERR_MC)   return "Media Changed";
    if (error & ATA_ERR_IDNF) return "ID Not Found";
    if (error & ATA_ERR_MCR)  return "Media Change Request";
    if (error & ATA_ERR_ABRT) return "Command Aborted";
    if (error & ATA_ERR_TK0NF) return "Track 0 Not Found";
    if (error & ATA_ERR_AMNF) return "Address Mark Not Found";
    return "Unknown Error";
}
```

---

## 參考資料

- [OSDev Wiki - ATA PIO Mode](https://wiki.osdev.org/ATA_PIO_Mode)
- [OSDev Wiki - Partition Table](https://wiki.osdev.org/Partition_Table)
- ATA/ATAPI-8 Specification (T13/1699-D)
