# DSP-Assignment-3
### 學號:711481109 姓名:莊祐儒

---
作業包含以LCCDE實現的Low-Pass Filter來轉換音檔的Sampling Rate的程式碼實現及分析以及FIR low-pass filter的作圖。

---


###  **LCCDE 低通濾波取樣率轉換(以C語言撰寫)**

## 目錄

1.標頭與常數定義

2.WAV 檔案結構定義

3.WAV 檔案讀取函數（read_wav_stereo）

4.WAV 檔案寫入函數（write_wav_stereo）

5.FIR 濾波器設計函數（fir_design）

6.多相濾波器分解函數（polyphase_decompose）

7.多相取樣率轉換函數（src_polyphase）

8.主程式（Main Function）

9.總結


---

## 1. 標頭與常數定義

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

說明：

- `#include <stdio.h>`：提供檔案輸入輸出功能（如 fopen, fread, fwrite）。

- `#include <stdint.h>`：提供固定長度整數型別（如 int16_t），確保音訊資料位元數正確。

- `#include <math.h>`：提供數學函式（如 sin, cos），用於 FIR 濾波器設計。

- `#include <stdlib.h>`：提供動態記憶體配置（malloc, free）。

- `#include <string.h>`：提供字串與記憶體操作函式。

- `PI`：圓周率。

- `L` = 80：取樣率轉換中的內插倍率（upsampling factor）。

- `M` = 441：取樣率轉換中的抽取倍率（downsampling factor）。

- `P`= 1025：FIR 低通濾波器的 tap 數。

- `Wc` = PI / M：正規化截止角頻率。






## 2. WAV檔案結構定義

```c
typedef struct {
    char riff[4];
    uint32_t chunk_size;
    char wave[4];
    char fmt[4];
    uint32_t subchunk1_size;
    uint16_t audio_format;
    uint16_t num_channels;
    uint32_t sample_rate;
    uint32_t byte_rate;
    uint16_t block_align;
    uint16_t bits_per_sample;
    char data[4];
    uint32_t data_size;
} WAVHeader;
```

說明：

- `riff`：標示檔案為 RIFF 格式

- `chunk_size`：整個 WAV 檔案大小

- `wave`：標示音訊格式為 WAVE

- `fmt`：格式區塊識別字串 "fmt "

- `subchunk1_size`：格式區塊大小（PCM 通常為 16）

- `audio_format`：音訊格式（PCM 為 1）

- `num_channels`：聲道數（1 = 單聲道，2 = 立體聲）

- `sample_rate`：取樣率（Hz）

- `byte_rate`：每秒資料位元組數

- `block_align`：每個取樣框架的位元組數

- `bits_per_sample`：每個取樣的位元數

- `data`：資料區塊識別字串 "data"

- `data_size`：實際音訊資料大小






## 3. WAV 檔案讀取函數（read_wav_stereo）

```c
int read_wav_stereo(const char *filename, int16_t **L_buf, int16_t **R_buf, int *N, int *fs)
{
	FILE *fp = fopen(filename, "rb");      // Open WAV file in binary read mode
	
	// check wav file
	if(!fp) 
	{
		return -1;
	}
	
	WAVHeader h;                               // WAV file header structure
	fread(&h, sizeof(WAVHeader), 1, fp);        // Read WAV header from file
	
	
	// Check if the WAV file is stereo and 16-bit PCM
	if(h.num_channels != 2 || h.bits_per_sample != 16)
	{
		fclose(fp);
		return -1;
	}
	
	*fs = h.sample_rate;
	*N = h.data_size/4;
	
	
	*L_buf = (int16_t*)malloc((*N) * sizeof(int16_t));    // Allocate memory for left channel
    *R_buf = (int16_t*)malloc((*N) * sizeof(int16_t));    // Allocate memory for right channel
	
	
	//Read stereo samples
	int i;
	for( i = 0; i < *N; i++)
	{
		fread(&(*L_buf)[i], sizeof(int16_t), 1, fp);     
		fread(&(*R_buf)[i], sizeof(int16_t), 1, fp);     
	}
	
	fclose(fp);  //close file
	return 0;
}



```

說明：

- `fopen(filename, "rb")`：以二進位讀取模式開啟輸入 WAV 檔案。
若開檔失敗，回傳 -1。

- `fread(&h, sizeof(WAVHeader), 1, fp)`：讀取 WAV 標頭資料到 WAVHeader 結構。
  
- 格式檢查：
  - `h.num_channels != 2`：確認是否為立體聲。

  - `h.bits_per_sample != 16`：確認是否為 16-bit PCM。
   若不符合，關閉檔案 `fclose(fp)` 並回傳 -1。


- 計算與分配記憶體：

  - `*fs = h.sample_rate`：取得取樣率。

  - `*N = h.data_size / 4`：計算每個聲道的樣本數。

  - `*L_buf、*R_buf`：動態分配記憶體給左右聲道陣列。

- 讀取 PCM 音訊資料：

  - 使用 `for`迴圈依序讀取左、右聲道樣本。

- 關閉檔案：

  - `fclose(fp)`：關閉 WAV 檔案。

- 回傳值：

  - `return 0`: 成功回傳 0。



## 4. WAV 檔案寫入函數（write_wav_stereo）
 
 ```c
void write_wav_stereo(const char *filename, const int16_t *L_buf, const int16_t *R_buf, int N, int fs)
{
	FILE *fp = fopen(filename, "wb");          // Open file in binary write mode
	
	WAVHeader h = {
        {'R','I','F','F'},
        36 + N * 4,
        {'W','A','V','E'},
        {'f','m','t',' '},
        16, 1, 2, fs,
        fs * 4, 4, 16,
        {'d','a','t','a'},
        N * 4
    };
    
    fwrite(&h, sizeof(WAVHeader), 1, fp);  // Write WAV header to file
	
	
	//Write stereo samples
    int i;
	for(i = 0; i < N; i++)        
	{
		fwrite(&L_buf[i], sizeof(int16_t), 1, fp);  
		fwrite(&R_buf[i], sizeof(int16_t), 1, fp);   
	}
	
	fclose(fp);   //close file
}
```


說明：

- `fopen(filename, "wb")`：以二進位寫入模式開啟輸出檔案。

- 建立 `WAVHeader h`：設定 RIFF、格式區塊 (fmt) 及資料區塊 (data) 標頭。

- `fwrite(&h, sizeof(WAVHeader), 1, fp)`：將 WAV 標頭寫入檔案。

- `for` 迴圈：將左右聲道的 PCM 音訊資料分別寫入檔案。

- `fclose(fp)`：關閉檔案，完成寫入。





##  5. FIR 濾波器設計函數（fir_design）

```c
void fir_design(double *h)
{
	int m, n, mid = (P - 1) / 2;  // mid is filter center index
	double sum = 0;
	double sinc;
	
	for(n = 0; n < P; n++)
	{
		m = n - mid;
		if(m == 0)             //sinc center
		{
			sinc = Wc/PI;              
		}
		else
		{
			sinc = sin( Wc * m) / (PI * m);
		}
	    double w = 0.54 - 0.46 * cos(2 * PI * n / (P - 1));   //hamming window
	    h[n] = sinc*w;	      //impulse response
	}           
	     
    // Normalize filter coefficients    
	for(n = 0; n < P; n++)
	{
		sum += h[n];
	}
	
	for(n = 0; n < P; n++)
	{
		h[n] /= sum;
	}
}

```


說明:
- #### 1.計算濾波器中心索引 `mid = (P-1)/2`。

- #### 2.`for` 迴圈：

   - 計算對應的 `m = n - mid`。

   - 計算 sinc 函數：

     - 若 `m = 0`，取 `sinc = Wc / PI`（中心點值）。

     - 否則 `sinc = sin( Wc * m) / (PI * m)`。

   - 計算 Hamming window` w = 0.54 - 0.46 * cos(2 * PI * n / (P - 1))`。

   - 將 sinc 與窗函數相乘得到濾波器脈衝響應 `h[n] = sinc * w`。

- #### 3.正規化濾波器係數：

  - 先計算總和 `sum = Σ h[n]`。

  - 再將每個係數除以總和 `h[n] /= sum`，確保 DC gain = 1。


## 6. 多相濾波器分解函數（polyphase_decompose）

```c
void polyphase_decompose(const double *h, double h_poly[L][(P+L-1)/L], int *phase_len)
{
	int n, r, idx;
	int max_len = 0;  // maximum length among all polyphase filters
	for(r = 0; r < L; r++)
	{
		idx = 0;
		for(n = r; n < P; n+=L)        // Extract every L-th coefficient starting from index r
		{
			h_poly[r][idx] = h[n];
			idx++;
		}
		phase_len[r] = idx;     // Store the number of taps for this phase
		if(idx > max_len)
		{
			max_len = idx;
		}
	}
}
```

說明:
#### polyphase_decompose使用原因: 因為upsample時會再輸入之間差L-1個0，那因為convolution的運算是輸入和頻率響應相乘後再不斷累加，所以這些為0的值去做convolution基本上就是浪費時間和計算量，那為了解決這個問題才會使用polyphase_decompose，以跳過某些頻率響應的值以減少花費的時間與計算量。

- #### 1.初始化變數：

  - `max_len = 0`，用來記錄所有相位中最長的濾波器長度。

- #### 2.外層迴圈 `for(r = 0; r < L; r++)`：

  - 處理每個相位 r。

  - 初始化索引 `idx = 0`。

  - 內層迴圈 `for(n = r; n < P; n+=L)`：

    - 從原始 FIR 濾波器係數 h 中，每隔 L 個取一個值，對應到相位 r。

    - 將取出的係數存入 `h_poly[r][idx]`。

    - 索引值加一 `idx++`。

  - 將該相位的係數數量存入 `phase_len[r]`。

  - 更新最大長度 `max_len`（若此相位更長）。



## 7. 多相取樣率轉換函數（src_polyphase）

```c
void src_polyphase(const int16_t *x, int N_in, int16_t *y, int *N_out, double h_poly[L][(P+L-1)/L], const int *phase_len)
{
	int k, r, k0, x_idx;
	int n = 0;
	double acc;
	while(1)
	{
		r = (n * M) % L;            // Compute phase index (fractional part of n*M/L)
		k0 = (n * M - r) / L;       // Compute integer input sample index
		
		if(k0 >= N_in)
		{
			break;
		}
		
		acc = 0.0;  // Reset accumulator for current output sample
		
		
		for(k = 0; k < phase_len[r]; k++)  //convolution
		{
			x_idx = k0 - k;
			if(x_idx >= 0 && x_idx < N_in)
			{
				acc += x[x_idx] * h_poly[r][k];
			}
		}
		
		acc *=  (double)M / (double)L;   
		
		if(acc > 32767)       // Saturation to int16 range
		{
			acc = 32767;
		}
		if(acc < -32768)
		{
			acc = -32768;
		}
		
		y[n++] = (int16_t)acc;   // Store output sample and advance output index
	} 
	
	*N_out = n;
} 
```

說明:

### 多相 SRC 中 phase $r$ 與輸入索引 $k_0$ 的數學推導

考慮取樣率轉換比例為 $L/M$ 的多相 SRC，理想輸出可寫為：

$$y[n] = \sum_m h[m] x_{\uparrow}[nM - m]$$

其中 $x_{\uparrow}[n]$ 是上採樣 $L$ 倍後的訊號，滿足：

$$x_{\uparrow}[\ell] \neq 0 \iff \ell \equiv 0 \pmod{L}$$

也就是上採樣後訊號在非 $L$ 的倍數位置都是 0。

---

#### Step 1：找出對輸出有貢獻的濾波器係數 $m$

由於卷積中只有 $x_{\uparrow}[nM - m] \neq 0$ 的項才會對輸出有貢獻，因此必須滿足：

$$nM - m = Lk \implies m = nM - Lk, \quad k \in \mathbb{Z}$$

這一步是利用上採樣後訊號的零值，排除不必要的計算。

#### Step 2：對 $m$ 做 $L$ 分解（多相分解）

任何整數 $m$ 可唯一寫為：

$$m = k'L + r, \quad r \in \{0, 1, \dots, L-1\}$$

代入 Step 1 的結果 $m = nM - Lk$ 得：

$$k'L + r = nM - Lk \implies nM - r = (k + k') L$$

這一步把濾波器分解成 $L$ 個相位，每個 phase 有自己的一組濾波器係數。

#### Step 3：推出 Phase 指數 $r$

上式表示 $nM - r$ 必須能被 $L$ 整除，因此：

$$\boxed{r = (nM) \bmod L}$$

 $r$ 表示第 $n$ 個輸出樣本應該使用哪一個子濾波器分支。

#### Step 4：推出輸入索引基準 $k_0$

我們定義 $k_0 = k + k'$ 為對應原始輸入序列 $x[k]$ 的索引。由 $nM - r = Lk_0$ 可得：

$$\boxed{k_0 = \frac{nM - r}{L} = \left\lfloor \frac{nM}{L} \right\rfloor}$$

 $k_0$ 是計算第 $n$ 個輸出時，在輸入訊號中對齊的基準位置。


 ### 程式碼說明
- #### 1.初始化變數：

  - `n = 0`，輸出樣本索引。

  - `acc`，暫存累加值。

- #### 2.迴圈 `while(1)`：

  - 計算相位索引 `r = (n * M) % L`，對應於多相濾波器的哪一相。

  - 計算輸入樣本索引 `k0 = (n * M - r) / L`。

  - 若 `k0 >= N_in`，跳出迴圈，表示輸入已用完。

  - 初始化累加器 `acc = 0.0`。

  - 對當前相位的濾波器做卷積：

    - `for(k = 0; k < phase_len[r]; k++)`

       - 計算輸入索引 `x_idx = k0 - k`。

       - 若索引合法 `(0 <= x_idx < N_in)`，將輸入乘上濾波器係數累加到 acc。

  - 將累加值乘上比例 `(double)M / (double)L`，補償取樣率改變造成的增益。

  - 限幅到 `int16` 範圍 [-32768, 32767]。

  - 將累加結果存入輸出陣列 `y[n++]`。

- #### 3.將輸出樣本數存入 `*N_out`。

## 8. 主程式（Main Function）

```c

```
說明：

1.宣告輸入/輸出 PCM 陣列及取樣率、樣本數變數。

2.讀取輸入 WAV 檔案到左右聲道陣列。

3.設計 FIR 濾波器並做多相分解。

4.配置輸出陣列記憶體。

5.對左右聲道分別執行多相取樣率轉換（SRC）。

6.寫入轉換後的 WAV 檔案。

7.釋放記憶體並結束程式。




