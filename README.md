# DSP-Assignment-3
### 學號:711481109 姓名:莊祐儒

---
作業包含以LCCDE實現的Low-Pass Filter來轉換音檔的Sampling Rate的程式碼實現及分析以及FIR low-pass filter的作圖。

---


###  **LCCDE 低通濾波取樣率轉換(以C語言撰寫)**

## 目錄

1.標頭與常數定義

2.WAV 檔案結構定義

3.檢查命令列參數

4.開啟與讀取 WAV 檔頭

5.讀取 PCM 音訊資料

6.從檔名解析濾波 cutoff frequency

7.設定 RC 濾波參數

8.初始化濾波暫存

9.RC 低通濾波處理

10.寫出濾波後 WAV


---

## 1. 標頭與常數定義

```c
#include <stdio.h>
#include <math.h>
#include <memory.h>
#include <stdlib.h>
#include <string.h>
#define PI 3.14159265359
```

說明:
引入必要的標頭檔與常數：

- `stdio.h`, `stdlib.h`：檔案操作與動態記憶體分配。
- `math.h`：數學運算（例如 `round()`）。
- `string.h`：字串搜尋與擷取頻率。
- `PI`：供 RC 濾波公式使用。

## 2. WAV檔案結構定義

```c
typedef struct {
    char chunkID[4];     
    unsigned int chunkSize;
    char format[4];      
} RIFFHeader;

typedef struct {
    char subchunk1ID[4]; 
    unsigned int subchunk1Size; 
    unsigned short audioFormat; 
    unsigned short numChannels; 
    unsigned int sampleRate;    
    unsigned int byteRate;      
    unsigned short blockAlign;  
    unsigned short bitsPerSample;
} FmtSubchunk;

typedef struct {
    char subchunk2ID[4]; 
    unsigned int subchunk2Size;
} DataSubchunk;
```

說明：
這三個結構對應 WAV 檔案格式：

1. `RIFFHeader`：WAV 主標頭，包含 `"RIFF"` 與 `"WAVE"` 標誌。
2. `FmtSubchunk`：音訊格式資訊（取樣率、聲道數、位元深度等）。
3. `DataSubchunk`：音訊資料段標頭，描述資料長度。




## 3. 檢查命令列參數

```c
int main(int argc, char *argv[])
{
    if(argc != 3){
        printf("Usage: %s in_fn out_fn\n", argv[0]);
        return 1;
    }

    char *in_fn = argv[1];   
    char *out_fn = argv[2];  



```


說明：
確認執行時的輸入參數是否正確：

- `in_fn`：輸入 WAV 檔案名稱。
- `out_fn`：輸出 WAV 檔案名稱。
- 若參數不足，程式會提示正確使用方式後結束。



## 4. 開啟與讀取 WAV 檔頭
 
 ```c
    FILE *fp = fopen(in_fn, "rb");  // 以二進位模式開啟輸入 WAV
    if(!fp){
        fprintf(stderr, "Cannot open file %s\n", in_fn);
        return 1;
    }

    RIFFHeader riff;
    FmtSubchunk fmt;
    DataSubchunk data;

    fread(&riff, sizeof(RIFFHeader), 1, fp);
    fread(&fmt, sizeof(FmtSubchunk), 1, fp);
    fread(&data, sizeof(DataSubchunk), 1, fp);
```


說明：
以二進位模式開啟 WAV 檔案，依序讀取：

- RIFF 標頭：確認檔案格式。
- 格式資訊區（fmt chunk）。
- 音訊資料區（data chunk）標頭。





##  5. 讀取 PCM 音訊資料

```c
    int fs = fmt.sampleRate;
    int bitsPerSample = fmt.bitsPerSample;
    int numChannels = fmt.numChannels;
    size_t N = data.subchunk2Size / (numChannels * bitsPerSample / 8);

    short *stereo = malloc(N * fmt.numChannels * sizeof(short));
    fread(stereo, sizeof(short), N*fmt.numChannels, fp);
    fclose(fp);
```

說明：
從 fmt 區段中取得取樣率、通道數、位元深度。  
根據資料大小計算樣本數 `N`，並配置記憶體以讀取所有取樣。  
讀取完畢後關閉檔案。

## 6. 從檔名解析 cutoff frequency

```c
    int f = 0;  // cutoff frequency
    char *first_f = strchr(in_fn, 'f');
    if(first_f != NULL){
        char *second_f = strchr(first_f + 1, 'f');
        if(second_f != NULL){
            f = atoi(second_f + 1);
            printf("Extracted cutoff frequency f = %d Hz\n", f);
        }
    }
    if(f <= 0){
        fprintf(stderr, "Failed to extract cutoff frequency from filename.\n");
        return 1;
    }
```

說明：

透過字串搜尋函式 `strchr()` 找出第二個 `'f'` 之後的數字作為截止頻率。  
例如：輸入檔名 `"test_f400_in.wav"` → `f = 400 Hz`。  
若解析失敗則輸出錯誤訊息並結束。


## 7. 設定 RC 濾波參數

```c
    double T = 1.0/fs;   // 取樣週期
    double R = 1000;     
    double C = 1.0/(2*PI*f*1000);  // cutoff frequency 自動使用 f
    double a = R*C/(R*C+T);
```

說明：
設定 RC 濾波器參數：

- `T`：取樣週期 (1/fs)。
- `R`：電阻（固定 1kΩ）。
- `C`：依頻率計算電容值。
- `a`：離散化濾波係數。

## 8. 初始化濾波暫存

```c
    double out_l = 0, out_r = 0;
    size_t total_samples = data.subchunk2Size / (numChannels * bitsPerSample / 8);
```


說明：
初始化濾波暫存與計算總取樣數：

- `out_l`, `out_r`：左右聲道的上一個輸出值，用於一階 RC 濾波計算。
- `total_samples`：總取樣數 (每個通道的樣本數)，用於迴圈處理整個音訊資料。


## 9. RC 低通濾波處理

```c
    for (size_t n = 0; n < total_samples; n+=2){
        double in_L = stereo[n];       
        double in_R = stereo[n+1];     

        out_l = (1-a) * in_L + a * out_l;
        out_r = (1-a) * in_R + a * out_r;

        stereo[n] = (short)round(out_l);
        stereo[n+1] = (short)round(out_r);

        // 限幅避免溢位
        if (stereo[n] > 32767) stereo[n] = 32767;
        if (stereo[n] < -32768) stereo[n] = -32768;
        if (stereo[n+1] > 32767) stereo[n+1] = 32767;
        if (stereo[n+1] < -32768) stereo[n+1] = -32768;
    }

```
說明：
RC 低通濾波處理迴圈：

- 使用 `for` 迴圈逐樣本處理左右聲道 (`n+=2` 遍歷 stereo array)。
- `in_L`、`in_R`：當前左右聲道輸入樣本。
- RC 濾波公式：
- `out_l = (1-a) * in_L + a * out_l;`
  `out_r = (1-a) * in_R + a * out_r;`


其中 `a` 控制平滑程度，上一個輸出影響下一個輸出。
- 將濾波後結果回寫到 `stereo` 陣列，使用 `round()` 並轉型為 `short`。
- 限幅 (clipping)：確保輸出值不超過 16-bit PCM 的範圍 [-32768, 32767]，避免溢位。


## 10. 寫出濾波後 WAV

```c
    fp = fopen(out_fn, "wb");
    if(!fp){ fprintf(stderr, "Cannot save %s\n", out_fn); exit(1); }

    fwrite(&riff, sizeof(RIFFHeader), 1, fp);
    fwrite(&fmt, sizeof(FmtSubchunk), 1, fp);
    fwrite(&data, sizeof(DataSubchunk), 1, fp);
    fwrite(stereo, sizeof(short), N*fmt.numChannels, fp);

    free(stereo);
    fclose(fp);

    return 0;
}
```

說明：
寫出濾波後的 WAV 檔案：

- `fopen(out_fn, "wb")`：以二進位寫入模式開啟輸出檔案。
- 若開檔失敗，輸出錯誤訊息並結束程式。
- `fwrite`：
  - 先寫入 `RIFFHeader`、`FmtSubchunk`、`DataSubchunk` 標頭。
  - 再寫入濾波後的 PCM 音訊資料 `stereo`。
- 釋放動態配置的記憶體 `free(stereo)`。
- 關閉檔案 `fclose(fp)`。
- 程式結束 `return 0;`。
