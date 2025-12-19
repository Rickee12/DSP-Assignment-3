# DSP-Assignment-3
### 學號:711481109 姓名:莊祐儒

---
作業包含以LCCDE實現的Low-Pass Filter來轉換音檔的Sampling Rate的程式碼實現及分析以及FIR low-pass filter的作圖。

---




```c
#include <stdio.h>
#include <stdint.h>
#include <math.h>
#include <memory.h>
#include <stdlib.h>
#include <string.h>
#define PI 3.14159265359
#define L 80
#define M 441
#define P 1025
#define Wc  (PI/M)

```
