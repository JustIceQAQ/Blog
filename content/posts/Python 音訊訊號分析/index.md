---
title: 'Python 音訊訊號分析'
date: '2026-10-07T09:08:41+08:00'
draft: false
tags: [ "Python", "pydub", "auditok", "librosa" ]
categories: [ "音訊處理", "Package", "踩地雷" ]
---

{{< alert icon="fire" cardColor="#e63946" iconColor="#1d3557" textColor="#f1faee" >}} 本文部分內容與程式碼皆已過時。
~~因為筆者懶的更新~~
{{< /alert >}}

## 來自未來的呼喚

----

- 針對財政部的稅籍登記資料公示查詢，時至今日已經有API可以串了
  (~~雖然資訊仍舊不完整，比起API查詢結果，UI查詢結果仍舊更完整~~)，因此沒有特別的事情，也別再試圖攻破網站的圖形驗證碼了。
    - `https://eip.fia.gov.tw/OAI/api/businessRegistration/{ubn}`
    - `ubn`: 企業統一編號

## 前言

----

- 由於寫Side Project，想把財政部的 **稅籍登記資料公示查詢**[^稅籍登記資料公示查詢-財政部稅務入口網]給串接起來。

- 畫面如下所示，該頁面可為輸入 **統一編號**與 **驗證碼**後，並按下確認送出按鈕，即完成查詢
    - ![](https://i.imgur.com/nI7AZ7E.png)

- 問題主要出在這一欄資料，這類圖形驗證碼Captcha把大多數爬蟲給擋了下來，導致網站...不至於收到大量機器人查詢。
    - ![](https://i.imgur.com/Sbt7r8n.png)

- 基於政府開放資料，其實也有找到財政部財政資訊中心提供的全國營業 (稅籍)登記資料集[^全國營業稅籍登記資料集]
  開放資料，因此可以串接下載網址後每日更新local的資料集，並再做後續應用。

- 但整個『登記資料集.zip』就有 58.3MB，解壓縮後的 CSV 檔更高達 283MB。光是設定 Cron Job 每天下載，倘若服務部署於雲端，就準備流量費付到飽吧。
    - ![](https://i.imgur.com/h2eKd5z.png)
- 另外針對圖形驗證碼破解的部分，網路上已經有很多教學文，不外乎是先:
    - 找到目標網站的圖形驗證碼後
        - 如果圖形驗證碼夠簡單
            - 那OCR一下就完工
        - 如果圖形驗證碼夠複雜，你可能要
            - 蒐集目標網站的圖形驗證碼資料 (1w+張圖片起跳)
            - 資料前處理，幫1w+張驗證碼上好正確答案
            - 資料前處理，要先確定N位數的圖形驗證碼該如何切割
            - 使用CNN深度學習開始訓練/測試模型


- 到此...總結一下，如果要解決網站圖形驗證碼的問題，此刻有以下方法：
    - CNN 圖形驗證碼
        - 自己收資料、自己訓練模型、自動預測答案
        - 我自己已體驗過。光是收集完1w+張圖片，目標網站就發現大量圖形驗證碼請求，圖形驗證碼樣式就換了 (嗯嗯...功虧一簣)
    - 使用政府開放資料
        - 定時下載，用網路流量、儲存空間換取時間
        - 我自己只下載過一次，之後就不想用這招了
    - ~~放棄~~

## 動機與目的

----

- 我在思考對策時，我突然看到了圖中紅框的播放icon，並且按了下去
    - ![](https://i.imgur.com/KxurmE1.png)
    - ![](https://i.imgur.com/n5JGYiH.png)
    - <audio src="sqcsf9.wav" controls onloadedmetadata="this.volume=0.4">內容為sqcsf9的音訊</audio>
- 我就開始思考，比起 CNN 圖形驗證碼訓練得要死要活；政府開放資料檔案又大又要等下載。我們人耳聽到這段音訊後，就知道答案是「sqcsf9」，那為何不直接讓Python「聽」完後，直接判斷答案就好?
- 於是就開始了本次的音訊訊號分析之旅

## 實作過程

----

- 過程大致分為:
    - 檔案轉換 (mp3 to wav)
    - 音訊切割
    - 音訊相似度比對

### 檔案轉換 (mp3 to wav)

----

```python=
import pydub
# Windows 需下載ffmpeg檔案
pydub.AudioSegment.converter = r"C:\Anaconda3\Library\ffmpeg-2022-10-27-git-00b03331a0-full_build\bin\ffmpeg.exe"
pydub.AudioSegment.ffprobe   = r"C:\Anaconda3\Library\ffmpeg-2022-10-27-git-00b03331a0-full_build\bin\ffprobe.exe"
sound = pydub.AudioSegment.from_mp3("sqcsf9.mp3")
sound.export(f"sqcsf9.wav", format="wav")
```

### 音訊切割

----

```python=
import auditok
region = auditok.load("sqcsf9.wav")
region.plot()
```

- 音訊頻譜圖
    - ![](https://i.imgur.com/1HYch2Z.png)

```python=
audio_regions  = region.split_and_plot(energy_threshold=70)
print(audio_regions)
# [AudioRegion(duration=0.450, sampling_rate=22050, sample_width=2, channels=1),
# AudioRegion(duration=0.500, sampling_rate=22050, sample_width=2, channels=1),
# AudioRegion(duration=0.550, sampling_rate=22050, sample_width=2, channels=1),
# AudioRegion(duration=0.450, sampling_rate=22050, sample_width=2, channels=1),
# AudioRegion(duration=0.550, sampling_rate=22050, sample_width=2, channels=1),
# AudioRegion(duration=0.400, sampling_rate=22050, sample_width=2, channels=1)]
```

- 音訊分割頻譜圖 (分別代表驗證碼 s q c s f 9 六位音訊頻譜圖)
    - ![](https://i.imgur.com/w9b6oB6.png)

```python=
# 將音訊分割後個別儲存成單一音訊模型
for a, name in zip(audio_regions, "sqcsf9"):
    a.plot()
    a.save(f"{name}.wav")
```

- `sqcsf9`切割後的個別音訊頻譜圖與音訊
    - `s` 頻譜圖
        - ![](https://i.imgur.com/uEBQVs8.png)
        - <audio src="s.wav" controls onloadedmetadata="this.volume=0.4">內容為s的音訊</audio>

    - `q` 頻譜圖
        - ![](https://i.imgur.com/TCJRPQN.png)
        - <audio src="q.wav" controls onloadedmetadata="this.volume=0.4">內容為q的音訊</audio>

    - `c` 頻譜圖
        - ![](https://i.imgur.com/NrnXzwg.png)
        - <audio src="c.wav" controls onloadedmetadata="this.volume=0.4">內容為c的音訊</audio>

    - `s` 頻譜圖
        - ![](https://i.imgur.com/uEBQVs8.png)
        - <audio src="s.wav" controls onloadedmetadata="this.volume=0.4">內容為s的音訊</audio>

    - `f` 頻譜圖
        - ![](https://i.imgur.com/PGVdtHi.png)
        - <audio src="f.wav" controls onloadedmetadata="this.volume=0.4">內容為f的音訊</audio>

    - `9` 頻譜圖
        - ![](https://i.imgur.com/EUj8e5E.png)
        - <audio src="9.wav" controls onloadedmetadata="this.volume=0.4">內容為9的音訊</audio>

### 音訊相似度比對

----

- 音訊本質上是時間序列資料。即使兩段聲音在人耳聽來相同，只要起始位置（相位）或取樣時間點略有偏移，其波形的取樣數值就會不同。
- 以驗證碼 `sqcsf9` 為例，第一位 `s` 與第三位 `s` 聽起來是同一個字母，但我將兩段音訊不經傅立葉轉換、直接在時域做餘弦相似度比對，結果兩者被判定為「不相似」。
- 因此需要先做時域到頻域的轉換。使用傅立葉轉換會把音訊拆解成多個不同頻率、不同振幅的弦波疊加，使比對的重點從「波形在時間上的位置」轉為「聲音包含哪些頻率成分」，對時間偏移較不敏感。
- 本次實作流程以 `librosa` 讀取音訊，透過短時距傅立葉轉換（STFT）與快速傅立葉轉換（FFT）取得頻域特徵，再以餘弦相似度比較當下音訊與各字母樣本音訊，最後選出相似度最高的字母作為辨識結果。

```python=
import librosa
import numpy as np
class LibrosaAudioAnalyzer:
    def __init__(self, filename, fft_size=256, input_sr=41000):
        y, sr = librosa.load(filename, sr=input_sr)
        self.y = y
        self.sr = sr
        self.fft_size = fft_size
        self.filename = filename

    def get_spectrum(self) -> np.array:
        data = np.abs(librosa.stft(self.y, n_fft=self.fft_size))
        bins = librosa.fft_frequencies(sr=self.sr, n_fft=self.fft_size) / 1000
        average_amplitude = data.mean(axis=1)
        return np.dstack((bins, average_amplitude))
    
    def plot_spectrum(self, min_freq=0, max_freq=None, fill=False, title="Spectrogram - Average Frequency"):       
        target_np = self.get_spectrum()[0]
        plt.plot(target_np[:, 0], target_np[:, 1], color='g')
        plt.xlabel("Frequency (kHz)", fontsize=20)
        plt.ylabel("Amplitude (scaled)", fontsize=20)
        plt.title(self.filename, fontsize=26)
        plt.legend() 
        plt.show()

```

```python=
target = LibrosaAudioAnalyzer("runtime_3.wav", input_sr=44100, fft_size=44100)
wav_model = LibrosaAudioAnalyzer("s.wav", input_sr=44100, fft_size=44100)
target.plot_spectrum()
wav_model.plot_spectrum()

target
# array([[[0.00000000e+00, 3.37956095e+00],
#         [1.00000000e-03, 3.40951300e+00],
#         [2.00000000e-03, 3.34269953e+00],
#         ...,
#         [2.20480000e+01, 1.47242602e-02],
#         [2.20490000e+01, 1.47262774e-02],
#         [2.20500000e+01, 1.47233009e-02]]])

wav_model
# array([[[0.00000000e+00, 3.37956095e+00],
#         [1.00000000e-03, 3.40951300e+00],
#         [2.00000000e-03, 3.34269953e+00],
#         ...,
#         [2.20480000e+01, 1.47242602e-02],
#         [2.20490000e+01, 1.47262774e-02],
#         [2.20500000e+01, 1.47233009e-02]]])
```

- ![](https://i.imgur.com/XXTlOBf.jpg)

```python=
# 餘弦相似度(cosine similarity)
def cosine_similarity(x, y):
    return np.dot(x, y) / (np.linalg.norm(x) * np.linalg.norm(y))

target_np = np.squeeze(target.get_spectrum().reshape(1, -1))
wav_model_np =  np.squeeze(wav_model.get_spectrum().reshape(1, -1))
cosine_similarity(target_np, wav_model_np)
# 0.9999999999999999
```

## 結論

----

- 透過上述二音訊相似度比對，可以找出該片段音訊最為相似的音訊字母，最後能成功比對出驗證碼音訊的所有字母。

[^稅籍登記資料公示查詢-財政部稅務入口網]: https://www.etax.nat.gov.tw/etwmain/etw113w1/ban/query
[^全國營業稅籍登記資料集]: https://data.gov.tw/dataset/9400


{{< alert icon="fire" cardColor="#e63946" iconColor="#1d3557" textColor="#f1faee" >}} 本文部分內容與程式碼皆已過時。
~~因為筆者懶的更新~~
{{< /alert >}}