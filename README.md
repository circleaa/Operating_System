# Multithreaded Arithmetic Expression Evaluator (多執行緒四則運算解析器)

![C++](https://img.shields.io/badge/C++-00599C?style=flat-square&logo=c%2B%2B&logoColor=white)
![Pthreads](https://img.shields.io/badge/Pthreads-POSIX-4B0082?style=flat-square)
![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420?style=flat-square&logo=ubuntu&logoColor=white)

## Introduction
本專案為作業系統層級之並行處理 (Concurrent Programming) 實作。透過使用 POSIX Threads (Pthreads) API 開發多執行緒系統，針對複雜的 Infix 四則運算式進行並行的 Prefix 與 Postfix 轉換及計算。

本專案重點在於**執行緒生命週期管理 (Thread Lifecycle Management)**、**無 IPC 的資料回傳機制**，以及支援高達 25 位數之**大數運算 (Big Integer Computation)**。

## System Architecture & Algorithm

系統架構由單一 Main Thread 與兩個 Work Threads 構成：

1.  **Main Thread (調度與監控)：**
    *   負責讀取輸入檔案，解析 Infix 運算式。
    *   動態生成 (Spawn) 兩個子執行緒 (`pthread_create`)。
    *   回收資源 (`pthread_join`) 並接收子執行緒回傳的執行時間數據，最後進行系統效能分析 (Wall time量測)。
2.  **Work Thread 1 (Prefix 解析器)：**
    *   將 Infix 運算式轉換為 Prefix 形式，處理括號優先級與 associativity (如 left-to-right 運算)。
    *   實作 25 位數的大數運算器進行精準計算，處理異常錯誤並回傳執行時間 (ms)。
3.  **Work Thread 2 (Postfix 解析器)：**
    *   將 Infix 運算式轉換為 Postfix 形式並進行大數運算，量測並回傳獨立的執行效能。

## Usage

本專案開發與測試環境為 Ubuntu 24.04+ (64-bit)，並使用 g++ (13.2+) 編譯。

```bash
# 1. 編譯程式碼 (必須連結 pthread 函式庫)
g++ -o prog2 main.cpp -lpthread

# 2. 執行程式 (傳入資料檔名)
./prog2 prog2data.txt
