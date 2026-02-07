# 🛠️ Project: GhostOS (Minimalist Bottom-Up Exploration)

### 專案願景

**「不重造輪子，但拆解輪子的構造。」**
這是一個以「思維導圖」與「邏輯驗證」為主的 Side Project。我們透過 VS Code 組織 OS 底層知識，跳過冗長的 C/Assembly 編寫，直接切入硬體與軟體的交界處。

---

## 📂 專案目錄結構

我們將模擬一個真實 OS 的 Source Tree，但資料夾內放的是「邏輯拆解」與「關鍵暫存器定義」。

* `/docs`：核心架構設計書 (如：記憶體佈局、中斷處理表)。
* `/boot`：引導扇區邏輯與 CPU 模式切換 (Real Mode -> Protected Mode)。
* `/kernel`：內核核心邏輯 (Memory Management, Scheduler)。
* `/drivers`：硬體抽象層 (Keyboard, Screen, Disk)。
* `/references`：收集 Intel 手冊或 xv6 源碼的片段分析。

---

## 🧩 第一階段：硬核知識攻堅目標

### 1. 啟動區塊 (The Bootloader)

* **目標**：理解 CPU 通電後如何從 `0xFFFFFFF0` 位址開始執行第一條指令。
* **重點知識**：
* MBR (Master Boot Record) 的 512 Bytes 限制。
* **A20 Gate** 的歷史包袱與開啟方式。
* 進入 **GDT (Global Descriptor Table)** 的跳轉。



### 2. 記憶體虛擬化 (Memory Paging)

* **目標**：理解為什麼 `malloc` 拿到的位址是假的。
* **重點知識**：
* CR3 暫存器與多級頁表 (Page Tables)。
* **TLB 緩存** 如何加速位址轉換。
* 理解 **Page Fault** 如何驅動現代 OS 的「懶加載」機制。



### 3. 特權級別與中斷 (Ring 0 & Interrupts)

* **目標**：理解 `syscall` 怎麼讓應用程式安全地要求硬體服務。
* **重點知識**：
* **IDT (Interrupt Descriptor Table)** 的結構。
* CPU 特權級別切換 (Ring 3 to Ring 0)。
* 系統調用號與暫存器參數傳遞。



---

## 🛠️ VS Code 推薦配置

為了達成「最少精力學習」，建議安裝以下插件：

1. **Markdown All in One**：自動生成目錄，方便導覽複雜的技術文檔。
2. **Draw.io Integration**：直接在 VS Code 繪製分頁表結構或中斷流程圖。
3. **Hex Editor**：用來看二進制文件（當我們真的想看 MBR 長什麼樣子時）。

---

## 📝 當前待辦 (Roadmap)

* [ ] 撰寫第一份技術筆記：`boot/bootloader_logic.md` (分析 BIOS 到 Kernel 的跳轉)。
* [ ] 繪製記憶體分頁映射圖：`kernel/memory_layout.drawio`。
* [ ] 模擬設計一個 `Syscall` 流程：當 App 想要 Print "Hello" 時，硬體發生了什麼。

---
