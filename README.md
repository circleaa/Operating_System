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
<img src="multithreading _images/normal_calc.png" width="300">
<img src="multithreading _images/bignum_calc.png" width="300">

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
```

## Source Code
[完整程式碼](./prog2.cpp)

```cpp
#include <iostream>      // 基本輸出入功能
#include <fstream>       // 檔案讀取用
#include <sstream>       // 字串流處理
#include <stack>         // 用於 prefix/postfix 計算堆疊
#include <queue>         // 用於儲存中序轉 postfix/prefix 的 token
#include <string>        // 處理字串
#include <pthread.h>     // Pthreads API
#include <sys/time.h>    // gettimeofday 取得時間
#include <unistd.h>      // POSIX 系統函數
#include <cstdlib>       // exit()
#include <cmath>         // 數學函數
#include <iomanip>       // 控制輸出格式（小數點）
#include <gmpxx.h>       // GMP 的 C++ 版本（mpz_class）

using namespace std;

struct ThreadData {
    string infix;                // 中序運算式
    string result_expr;          // prefix/postfix 運算式字串
    string result_value_str;     // 最終運算結果（以 string 形式表示）
    double exec_time_ms;         // 執行所需時間（毫秒）
    pthread_t tid;               // 執行緒 ID
};

double current_time_ms() { // 取得目前時間
    struct timeval tv;
    gettimeofday(&tv, nullptr);
    return tv.tv_sec * 1000.0 + tv.tv_usec / 1000.0;
}

int precedence(char op) { // 運算子優先順序設定
    if (op == '+' || op == '-') return 1;
    if (op == '*' || op == '/') return 2;
    return 0;
}

bool isLeftAssoc(char op) { return true; } // 所有運算子皆為左結合
bool isOperator(char c) { // 判斷是否為運算子字元
    return c == '+' || c == '-' || c == '*' || c == '/'; }

bool isBalancedParentheses(const string& expr) { // 確認括號是否成對
    int balance = 0;
    for (char c : expr) {
        if (c == '(') balance++;
        else if (c == ')') balance--;
        if (balance < 0) return false;
    }
    return balance == 0;
}
// 將 infix 運算式轉為 postfix，並產生字串與 queue
queue<string> infixToPostfix(string expr, string& postfixStr) {
    stack<char> opStack; //存放運算符
    queue<string> output; //儲存轉換後的後序表示式的元素
    postfixStr.clear();
    for (size_t i = 0; i < expr.length();) {
        if (isdigit(expr[i])) { // 處理數字
            string num;
            while (i < expr.length() && isdigit(expr[i])) {
                num += expr[i++];
            }
            output.push(num);
            postfixStr += num + " "; //在其後添加空格
        } else if (expr[i] == '(') { //如果當前字符是左括號 '('，將其推入 opStack
            opStack.push(expr[i++]);
        } else if (expr[i] == ')') { //如果當前字符是右括號 ')'
            while (!opStack.empty() && opStack.top() != '(') {
                postfixStr += opStack.top(); postfixStr += " ";
                output.push(string(1, opStack.top())); //將運算符添加到 postfixStr 和 output 佇列中，直到遇到左括號 '('
                opStack.pop();
            }
            if (!opStack.empty() && opStack.top() == '(') opStack.pop();
            else { cerr << "Error: mismatched parentheses" << endl; exit(1); }
            i++;
        } else if (isOperator(expr[i])) { //處理運算符
            while (!opStack.empty() &&  //當堆疊不空且運算符的優先級低於或等於堆疊頂部的運算符
                   ((isLeftAssoc(expr[i]) && precedence(expr[i]) <= precedence(opStack.top())))) {
                postfixStr += opStack.top(); postfixStr += " "; //彈出堆疊頂部的運算符
                output.push(string(1, opStack.top()));
                opStack.pop();
            }
            opStack.push(expr[i++]);
        } else {
            cerr << "Error: invalid character '" << expr[i] << "'" << endl;
            exit(1);
        }
    }
    while (!opStack.empty()) {
        if (opStack.top() == '(' || opStack.top() == ')') {
            cerr << "Error: mismatched parentheses" << endl;
            exit(1);
        }
        //彈出堆疊中的運算符並將其加入 postfixStr 和 output 佇列
        postfixStr += opStack.top(); postfixStr += " ";
        output.push(string(1, opStack.top()));
        opStack.pop();
    }
    return output;
}
// infix 轉 prefix：反向掃描法 + stack 模擬
queue<string> infixToPrefix(string expr, string& prefixStr) {
    stack<string> operands; //存放操作數（數字）
    stack<char> operators; //存放運算符
    for (int i = expr.length() - 1; i >= 0; --i) {
        char c = expr[i]; //使用反向掃描的方式來遍歷中序表達式 expr，從右到左處理每一個字符
        if (isspace(c)) continue; //當前字符是空白字符，則跳過
        if (isdigit(c)) { //處理數字
            string num(1, c);
            while (i - 1 >= 0 && isdigit(expr[i - 1])) {
                num = expr[--i] + num;
            } //將數字字符添加到 num 中，直到不再是數字為止
            operands.push(num);
        } else if (c == ')') { //右括號 ')'，則將其推入 operators 堆疊中
            operators.push(c);
        } else if (c == '(') { //左括號 '('
            while (!operators.empty() && operators.top() != ')') {
                string a = operands.top(); operands.pop();
                string b = operands.top(); operands.pop();
                char op = operators.top(); operators.pop();
                operands.push(string(1, op) + " " + a + " " + b);
            } //每次彈出兩個操作數和一個運算符，並生成一個新的前序表達式
            if (!operators.empty()) operators.pop();
        } else if (isOperator(c)) { //字符是運算符（+、-、*、/）
            //檢查當前運算符的優先級是否小於堆疊頂部運算符的優先級
            while (!operators.empty() && precedence(c) < precedence(operators.top())) {
                string a = operands.top(); operands.pop();
                string b = operands.top(); operands.pop();
                char op = operators.top(); operators.pop();
                operands.push(string(1, op) + " " + a + " " + b);
            }
            operators.push(c);
        }
    }
    while (!operators.empty()) { //處理剩餘的運算符
        string a = operands.top(); operands.pop();
        string b = operands.top(); operands.pop();
        char op = operators.top(); operators.pop();
        operands.push(string(1, op) + " " + a + " " + b);
    }
    prefixStr.clear();
    queue<string> result;
    stringstream ss(operands.top());
    string tok;
    while (ss >> tok) {
        result.push(tok);
        prefixStr += tok + " ";
    }
    return result;
}
// 使用 GMP 的 mpz_class 計算 prefix 或 postfix
mpz_class evaluateGMP(const queue<string>& tokens, bool isPrefix) {
    vector<string> tokenList;
    queue<string> q = tokens;
    while (!q.empty()) { tokenList.push_back(q.front()); q.pop(); }
    stack<mpz_class> s;
    //設定 for 迴圈的起點、終點與步長
    int start = isPrefix ? tokenList.size() - 1 : 0;
    int end = isPrefix ? -1 : tokenList.size();
    int step = isPrefix ? -1 : 1;
    for (int i = start; i != end; i += step) {
        string token = tokenList[i];
        if (isdigit(token[0])) { //判斷開頭是數字
            s.push(mpz_class(token));
        } else {
            if (s.size() < 2) { cerr << "Error: insufficient operands" << endl; exit(1); }
            mpz_class a = s.top(); s.pop();
            mpz_class b = s.top(); s.pop();
            mpz_class res;
            if (token == "+") res = isPrefix ? a + b : b + a;
            else if (token == "-") res = isPrefix ? a - b : b - a;
            else if (token == "*") res = isPrefix ? a * b : b * a;
            else if (token == "/") {
                if ((isPrefix ? b : a) == 0) { cerr << "Error: division by zero" << endl; exit(1); }
                res = isPrefix ? a / b : b / a;
            }
            s.push(res);
        }
    }
    return s.top();
}

pthread_mutex_t lock; //互斥鎖
pthread_cond_t cond; //條件變數

void* threadPrefix(void* arg) {
    auto* data = (ThreadData*)arg; //將傳入的 void* 轉型為 ThreadData* 結構體指標
    double start = current_time_ms();
    string prefixStr;
    auto tokens = infixToPrefix(data->infix, prefixStr);
    mpz_class result = evaluateGMP(tokens, true);
    data->result_expr = prefixStr;
    data->result_value_str = result.get_str(); //將結果儲存回 ThreadData 結構中
    double end = current_time_ms();
    data->exec_time_ms = end - start;
    data->tid = pthread_self();
    cout << fixed << setprecision(2);
    cout << "[Child 1] " << prefixStr << "= " << data->result_value_str << " " << data->exec_time_ms << "ms" << endl;
    pthread_cond_signal(&cond); //向等待的 Child 2 發出條件變數訊號，通知它可以開始執行
    return nullptr;
}

void* threadPostfix(void* arg) {
    auto* data = (ThreadData*)arg;
    pthread_cond_wait(&cond, &lock); //等待 threadPrefix 使用 pthread_cond_signal() 發出的條件變數通知
    double start = current_time_ms();
    string postfixStr;
    auto tokens = infixToPostfix(data->infix, postfixStr);
    mpz_class result = evaluateGMP(tokens, false);
    data->result_expr = postfixStr;
    data->result_value_str = result.get_str();
    double end = current_time_ms();
    data->exec_time_ms = end - start;
    data->tid = pthread_self();
    cout << fixed << setprecision(2);
    cout << "[Child 2] " << postfixStr << "= " << data->result_value_str << " " << data->exec_time_ms << "ms" << endl;
    return nullptr;
}

int main(int argc, char* argv[]) {
    if (argc < 2) { cerr << "Usage: ./prog2 <filename>" << endl; return 1; }
    ifstream infile(argv[1]);
    if (!infile) { cerr << "Error: cannot open file." << endl; return 1; }
    string infix;
    if (!getline(infile, infix) || infix.empty()) {
        cerr << "Error: input file is empty or invalid." << endl;
        return 1;
    }
    if (infix.find_first_of(" \t\n") != string::npos) { //不能有空格或 tab
        cerr << "Error: expression should not contain whitespace." << endl;
        return 1;
    }
    for (char c : infix) { //字元必須是數字或操作符
        if (!isdigit(c) && string("+-*/()").find(c) == string::npos) {
            cerr << "Error: invalid character in expression." << endl;
            return 1;
        }
    }
    if (!isBalancedParentheses(infix)) { //括號要正確對應
        cerr << "Error: parentheses are not balanced." << endl;
        return 1;
    }
    if (infix.find('(') == string::npos && infix.find(')') == string::npos) { //檢查是否含有括號
        std::cout << "Error: no parentheses found in the expression." << std::endl;   
        return 1; 
    }
    cout << "The infix input: " << infix << endl;
    double main_start = current_time_ms();
    pthread_t t1, t2; //宣告兩個子線程 ID
    ThreadData d1{infix}, d2{infix}; //對應的輸入資料物件 d1、d2
    pthread_mutex_init(&lock, nullptr);
    pthread_cond_init(&cond, nullptr);
    pthread_create(&t1, nullptr, threadPrefix, &d1); //建立兩個子線程
    pthread_create(&t2, nullptr, threadPostfix, &d2);
    pthread_join(t1, nullptr); //主線程等待兩個子線程完成
    pthread_join(t2, nullptr);
    double main_end = current_time_ms();
    cout << fixed << setprecision(2);
    cout << "[Child 1 tid=" << d1.tid << "] " << d1.exec_time_ms << "ms" << endl;
    cout << "[Child 2 tid=" << d2.tid << "] " << d2.exec_time_ms << "ms" << endl;
    cout << "[Main thread] " << (main_end - main_start) << "ms" << endl;
    pthread_mutex_destroy(&lock);
    pthread_cond_destroy(&cond);
    return 0;
}
