# Multithreaded Arithmetic Expression Evaluator (多執行緒四則運算解析器)

![C++](https://img.shields.io/badge/C++-00599C?style=flat-square&logo=c%2B%2B&logoColor=white)
![Pthreads](https://img.shields.io/badge/Pthreads-POSIX-4B0082?style=flat-square)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white)

## Introduction
本專案為作業系統層級之並行處理 (Concurrent Programming) 實作。透過使用 POSIX Threads (Pthreads) API 開發多執行緒系統，針對複雜的 Infix 四則運算式進行並行的 Prefix 與 Postfix 轉換及計算。

本專案重點在於**執行緒生命週期管理 (Thread Lifecycle Management)**、**無 IPC 的資料回傳機制**，以及支援高達 25 位數之**大數運算 (Big Integer Computation)**。

[完整題目](./1132-cs305-prog2.pdf)

## Demo
執行範例
<img src="multithreading_images/小數運算.png" width="250">
<img src="multithreading_images/大數運算.png" width="250">

## System Architecture & Algorithm

*   **執行緒生命週期與同步：** Main thread 負責讀取檔案並生成兩個 Work threads。內部實作 `pthread_mutex_t` 與 `pthread_cond_t`，確保 Thread 1 必然於 Thread 2 之前完成輸出，避免 I/O 競爭 (Race Condition)。
*   **大數並行運算：** 透過 C++ GMP 封裝庫 (`<gmpxx.h>`)，突破標準整數限制，支援高達 25 位數之運算，且支援負數運算結果。
*   **嚴謹的例外處理 (Robustness)：** 內建多重防呆機制，包含：排除運算式空格、攔截不合法字元、檢查不成對括號，以及防止除以零之錯誤。
*   **精準效能量測：** 運用時鐘函數精準計算各執行緒之 Wall time (實際消逝時間)，並擴充至小數點下兩位數以利進行微秒級效能比較。

## Environment & Usage
本專案於 Windows Subsystem for Linux (WSL) 環境下使用 VS Code 開發。

### 1. 安裝依賴函式庫 (GMP)
```bash
#支援大數運算
sudo apt-get update
sudo apt install libgmp-dev libgmpxx41dbl
```

### 2. 編譯執行
```bash
#編譯
g++ prog2.cpp -lgmpxx -lgmp -lpthread -o prog2  //-lgmpxx 是 GMP 的 C++ 封裝函式庫 
#執行
echo "1+2*4+(7-5)/2" > prog2.data  //建立 prog2.data檔案
./prog2 prog2.data  //執行
