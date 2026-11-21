# HFSS 手動模擬學習手冊：本 repo 的操作流程

> 這是一份獨立的學習筆記，不是 `skill/` 的正式規範，也不取代各 `DesignN/README.md`、
> `design.json` 或該 Design 的既有 Python entry point。它的用途是讓你在 Ansys Electronics
> Desktop (AEDT) 的 HFSS 介面中，能看懂、檢查、手動設定與重跑本 repo 已定義的模擬。
>
> **最重要的原則：不要把 GDS 匯入 HFSS 當作本 repo 的模擬幾何。** HFSS 應使用各 Design 的
> Qiskit Metal `build_design()` 所建立的 *simulation geometry*。GDS 是版圖紀錄／製造輸出；
> 它不包含 Qiskit Metal 的 junction boundary、lumped inductance、open termination、port 定義，
> 也通常不等同於求解幾何。

## 1. 這個 repo 目前有哪些 HFSS 類型？

| 類型 | HFSS solver | 主要輸出 | 典型用途 |
|---|---|---|---|
| Resonator / readout eigenmode | Eigenmode | 複數模態頻率、mode field、近似 linewidth | 檢查共振頻率、辨認 mode、兩模反推 \(f_p,f_r,J,\kappa_p\) |
| Driven feedline / S-parameter | Driven Modal | 複數 \(S_{11}, S_{21}\)、pole / zero fit | 讀出 dip、實際 \(J\)、\(\kappa\)、full-feed standing-wave 效果 |
| Flux-line mutual inductance | Driven Modal，低頻 Z-parameter | \(Z_{ij}(f)\)、\(M\)、\(I_{\Phi_0}\)、crosstalk | 驗證 flux line 對 SQUID / fluxonium loop 的耦合 |
| Linear qubit / coupler mode sweep | Eigenmode | 多個 \(L_J\) 下的 linear mode | 配合 Q3D prior 擬合線性 \(C^{-1}\)，得到 \(E_C\)、charge-unit coupling |
| Qubit-readout eigenmode + EPR | Eigenmode + pyEPR | \(f_q\)、readout modes、\(\alpha\)、\(\chi\)、EPR | 含 linear JJ 模型的 qubit / readout cross-check |

Q3D capacitance extraction 不是 HFSS；請另看 `skill/cap_matrix_analysis.md`。HFSS 的線性頻率也
不是非線性 qubit \(f_{01}\)，更不會直接產生 \(E_J\) 或 \(E_L\)。這些是 layout / user / quantum-model
inputs。

## 2. 開始前：選定一個可追溯的 case

在 AEDT 按下 Analyze 前，先寫下以下內容。這個步驟避免「看起來跑完了，但不知道跑的是哪一版」。

1. 選定 `DesignN`，讀其 `README.md` 和 `design.json`。
2. 選定 geometry revision、要渲染的 components，以及上游 Q3D / circuit-model run。
3. 記下 simulation geometry 與 fab GDS 的差別；HFSS 使用前者，不加 fabrication hooks、cheesing 或
   e-beam JJ drawing。
4. 為本次工作建立新的 run folder。新 run 應使用 `code/general/run_record.py`，內容至少包括：
   command、time/timezone、working directory、AEDT / Python version、geometry revision、input hash、
   upstream run、solver setting、raw data 與 convergence evidence。
5. 在 Design 既有的 `simulation/aedt/<Design>.aedt` project 開新 HFSS design；不要覆寫已存在的
   design。`code/general/save_aedt.py` 的原則是一個 Design 共用一個 AEDT project、每次 run 有獨立
   design name。

### 2.1 環境與硬體檢查

* 使用 repo 指定的 `qmetal` conda environment 和 AEDT 2021 R2；不要改用 system Python 或另一套
  Qiskit Metal。
* AEDT 先啟動，再從 Python renderer 建立 native AEDT design。這可保留正確的 substrate、air box、
  nets、lumped junction、open termination 及 port sheets。
* HFSS scratch disk 保留至少 80 GB。大型 run 的 `hfss_*.pjt` 可佔 20–45 GB；full-chip run 甚至可能
  超出 24–32 GB RAM。
* 設定 scratch directory 到有足夠空間的磁碟。可先以 `code/general/hfss_dirs.py` 檢查；不要在 solver
  正在跑時改 temp directory。
* 先確認 layout 本身已通過基本 DRC：無短 segment、pin direction 正確、無 crossing、元件 ground strip
  滿足規則。HFSS 不會替你修正壞的幾何。

## 3. 共用的 HFSS 建模規則

### 3.1 應從 native AEDT design 開始，而非重畫或匯入 GDS

正確順序是：

```text
Design layout/build_design()
    -> Qiskit Metal HFSS renderer
    -> native AEDT design（geometry、ports、lumped elements、boundaries）
    -> 在 AEDT GUI 中檢查 / 設定 / Analyze / export
```

若要手動操作，最實用的做法是先執行目標 Design 現有 HFSS script 的 dry-run、render-only 模式，或在
script 渲染後暫停；再在 AEDT GUI 中開啟該 native design。若 script 沒有這種模式，仍應複用其
`build_design()`、component list、`open_terminations`、`port_list`、junction settings 和 mesh target，
不要自行從 GDS 拼回模型。

### 3.2 幾何、net 與端點檢查

在 AEDT 的 3D Modeler / Project tree 逐項確認：

1. 只渲染本次問題需要的 components。局部 readout channel 不應不小心帶入無關全晶片金屬；全晶片
   case 則不得省略會改變 feedline 或 return path 的物件。
2. substrate material、厚度、box / buffer、ground plane 與所有 etch 是否來自同一 geometry revision。
3. CPW centre conductor、兩側 ground、island、pocket cutout 的 net 沒有誤合併或誤斷開。
4. 對每個 `ResEnd`、readout pin、charge pin、flux pin，確認它是 intended open、short 或 port。漏列
   open termination 常會把 \(\lambda/4\) 結構短路，結果完全失真。
5. Eigenmode / readout 的 junction 使用 layout 中的 lumped \(L_J\) input。不要把它解讀成 HFSS
   直接抽出的 \(E_J\)。

### 3.3 Ground bonds、PEC straps 與回流路徑

每次 HFSS run 都要檢查 ground continuity。

* 一般 etched CPW path：啟用 Qiskit Metal wire bonds，`wb_threshold='350um'`、`wb_offset='0um'`、
  `wb_size=2`；這會讓長線約每 300–350 µm 有 ground connection。
* 自動 bonds 不會覆蓋太短的 coupler stub、tee arm 或 hammer stub。這些位置要加入明確的 PEC strap。
* `PurcellReadout.strap_points()` 所列位置是必要 strap：兩個 hammer stub，加上每個 T 兩側的主線。
* 在 AEDT GUI 中目視檢查每條 strap 的兩端都接觸 ground。若 strap 懸空、與 signal 相交，或 auto-bond
  與手動 strap 重疊，先修正再跑。

缺少 straps 可能讓 ground return 繞路，造成非預期 filter–resonator coupling；這不是可以靠後處理
修掉的小誤差。

### 3.4 Mesh 與 solver 共通設定

本 repo 的標準是：

| 類型 | 標準 max pass | 必要初始 mesh |
|---|---:|---|
| Eigenmode | 6 | CPW、capacitor electrode、qubit electrode 皆設 length-based **10 µm** |
| Driven S-parameter | 4 | **明確**的 length-based mesh，預設 **10 µm**，不可只靠 automatic mesh |
| Low-frequency mutual inductance | 依 Design 的例外設定 | flux line、SQUID / loop 的關鍵金屬可用較細 mesh；必須做 pass-count comparison |

在 AEDT GUI 內，對目標金屬物件建立 `Mesh Operations -> Assign -> Length Based`，並記錄：mesh
operation name、選到哪些 object / face、最大長度，以及是否所有 CPW / electrode 都被涵蓋。對 driven
case，這不是可選優化，而是必要設定。

第一階 basis order 是預設。第二階 basis 常在第一 pass 就耗盡記憶體。Design 若因為 full-chip RAM
限制採用不同 pass 或 mesh，必須以該 Design README / notes 和 `run.json` 為準，不能把例外變成全域預設。

> Max pass 是執行預算，不是收斂證明。每個 run 都要保存最後 \(\Delta S\) 或 \(\Delta f\)、pass table，
> 並在需要 validation 時以較嚴格 / 較多 pass 的 run 比較目標物理量。

### 3.5 手動輸出與保存

每次 solve 後，至少匯出到該 run 的 `raw/`：

* setup 和 sweep setting 的截圖或文字 export；
* mesh operation、ports、open termination、wire bonds / straps 的證據；
* adaptive convergence table；
* complex eigenfrequency 或 complex S/Z data，不能只有 PNG；
* native `.aedt` snapshot（不必複製龐大的 `.aedtresults`）；
* summary 中的計算式、單位與限制。

不要將 `hfss_eig_f_convergence.csv`、隨機 screenshot 或 solver side file 寫到 repo root。

## 4. 流程 A：Resonator / readout Eigenmode

### 4.1 什麼時候使用

用於先定位 readout、Purcell filter、coupler 或 linear qubit mode。當 resonator 很窄、在 driven S21
看不到 dip，eigenmode 是先找 mode 的正確工具。

對一個 Purcell readout channel，通常渲染：feedline 的局部區段、\(C_p\)、filter、hammer \(C_j\)、
resonator、必要時 qubit 和 readout claw。兩端 50 Ω port sheet 會使 filter-like mode 帶有 loss，從而可
得到複數 eigenfrequency。

### 4.2 AEDT 手動設定步驟

1. **建立／選擇 Eigenmode design。** 確認 native design 是由目標 Design 的 simulation layout 渲染。
2. **確認 terminations。**
   - shorted filter / resonator end：保持 renderer 定義的 short；
   - resonator open end：必須是 open termination；
   - feedline 兩端：使用 50 Ω lumped / RLC port sheets，與自動 script 的 port 位置和方向一致。
3. **確認 junction。** qubit / coupler 必須使用 layout 所指定的 lumped \(L_J\)，junction sheet 兩端接到
   正確金屬。不要在同一 gap 額外建立矛盾的 port。
4. **建立 ground straps，檢查 wire bonds。** 見第 3.3 節。
5. **建立 10 µm length mesh。** 選所有本 case 的 CPW、capacitor 和 qubit electrode。對局部 case，
   不應漏掉 tee、hammer、claw 或窄電容區。
6. **建立 Eigenmode setup。**
   - `Minimum Frequency`：低於預期最低 mode；
   - `Number of Modes`：至少覆蓋需要辨認的 modes；典型 readout pair 取兩個，含 qubit 時取三個；
   - `Maximum Passes`：預設 6；
   - 使用 first-order basis；
   - 記錄 convergence target、minimum pass、minimum converged passes、percent refinement。
7. **Analyze。** 先看 adaptive pass 是否合理、記憶體是否暴增、bond / port 是否有 error。
8. **匯出複數 eigenfrequencies 與 convergence table。** 不只記 mode list 中的實部。
9. **mode assignment。** 以 E-field / current field、局部幾何分布、以及小幅改變某一 line length（例如
   約 20 µm）來辨認 filter-like / resonator-like / qubit-like mode；不可只照 mode 的排序編號。

### 4.3 由 eigenmode 讀取的量

若 AEDT 匯出的複數頻率以 GHz 表示為 \(z=f+i\gamma\)，且與本 repo 的慣例相同，則：

```text
f_mode / GHz       = Re(z)
kappa / 2π / MHz   = 2 × Im(z) × 1000
```

先確認符號慣例；不要不檢查就取絕對值。對 Purcell 兩模系統，兩個 complex roots 可反推 bare
\(f_p\)、\(f_r\)、\(J\)、\(\kappa_p\)，但要明確記錄假設：一個 lossy filter-like mode、一個較無損的
resonator-like mode、\(J\) 為實數。

Eigenmode 得到的 \(\kappa_p\) 常比 driven S21 / Q3D 估計高約 1.5–1.8 倍。因此它適合 mode
辨認和 cross-check；最終 external linewidth 優先用 driven S21。

### 4.4 Eigenmode 合格檢查

* 最後兩個 pass 的目標頻率差異是否足夠小；
* 目標 mode 的場形是否真的位於預期 resonator / filter，而非 box mode；
* 有 / 無 50 Ω port 的結果是否符合預期 loss 差異；
* 若改 20 µm filter arm，主要改 filter-like mode；改 resonator arm，主要改 resonator-like mode；
* geometry、ports、mesh 與 straps 是否都已寫入 run record。

## 5. 流程 B：Driven Modal S11 / S21

### 5.1 什麼時候使用

這是取得可量測 readout response 的主要流程。它用於：

* 取得 complex \(S_{11}\)、\(S_{21}\)；
* 找到 resonator / filter dips；
* pole fit 取得 \(f_p,f_r,J,\kappa_p\)；
* 檢查 series \(C_{in}\)、多 channel feedline、standing wave 與 tee placement；
* 對 launch pad 或兩埠小結構檢查 S-parameter。

### 5.2 AEDT 手動設定步驟

1. **從同一 simulation geometry 建立 Driven Modal design。** 若已有 matching eigenmode design，可用
   `Mesh Link -> <eigen design> : Setup : LastAdaptive`；HFSS 有時會靜默拒絕 import，必須確認
   `ImportMesh` 真的成功。
2. **建立兩個 50 Ω lumped ports。** 每張 port sheet 必須跨 signal conductor 與正確 ground，方向與
   feedline 的傳播方向一致。檢查 port sheet 沒有短到兩片 ground 或碰到錯誤 island。
3. **明確設定 opens / shorts。** 例如 Purcell 的 resonator open end 必須是 open；短路尾端應保留 short。
   不要依預設猜測。
4. **加入 straps / bonds 和明確 10 µm length mesh。** Driven case 必須在所有 CPW line 和 capacitor
   electrode 指派 explicit mesh。
5. **設定 adaptive solve frequency。** 選在目標 readout band 附近；記錄 `max_delta_s`、max/min pass、
   basis order 和 mesh selection。
6. **先跑 wide sweep。**
   - 頻率範圍要覆蓋預期 dip 的上下側；
   - 先以約 1 MHz 級別的 sampling 尋找 resonance；
   - 不要相信 interpolation 一定能找到很窄的 dip。
7. **再跑 fine sweep。**
   - 對每個 channel window 縮小頻帶；
   - 使用不大於 0.25 MHz 的 step；需要做 pole fit 時，25–50 kHz 級別通常較安全；
   - 保留 wide 和 fine 的 complex raw data。
8. **匯出 complex S data。** 至少含 frequency、S11、S21 的 real/imaginary parts，reference impedance、
   sweep type 和 points。不可只輸出 dB 圖。

### 5.3 S21 的分析順序

1. 在 wide sweep 找候選 dip，但不要把 dip index 當 mode identity。
2. 以每個 channel 的預期頻率範圍建立獨立 window。
3. 對 fine-band **complex** S21 做 pole / zero 或固定-zero fit；背景可使用低階 complex polynomial。
4. 本 repo 的 HFSS / ABCD 慣例為 \(e^{+j\omega t}\)。若沿用以 \(e^{-j\omega t}\) 寫的 fit，可能需要對
   S21 取 complex conjugate，否則 extracted \(\kappa\) 會出現錯號。
5. 只在兩個 hybrid dips 都解析且接近共振時，才把 \(J\) 視為受良好約束。遠失諧時通常只看得到
   \(J^2/\Delta\) 的小 shift。
6. 將 fit 和 HFSS raw curve 疊圖，保存 RMS residual、fit window、pole、zero、背景模型與初值。

### 5.4 何時可以引用 linewidth？

可引用的是明確寫成 \(\kappa/2\pi\) 或 FWHM 的 quantity，且須說明來源：

* Q3D \(C_1\) 的 \(Q_c\) 估計；
* driven S21 的 complex-pole fit；
* eigenmode complex frequency；
* hybrid mode FWHM。

這些不是自動相同的量。尤其 Purcell channel 的 \(\kappa_p\)、\(\kappa_r\) 與兩個 hybrid-mode linewidth
必須分開命名。含 series \(C_{in}\) 的 feedline 中，\(\kappa_p\) 會隨 tee 的電氣位置起伏，因此不可只
用孤立單 channel 的 hanger formula 當最終數值。

### 5.5 Driven case 的驗證

* final \(\Delta S\) 必須保存；broadband / well-damped case 通常以 0.05–0.10 作可接受範圍；
* 對 narrow、近乎無損的 resonance，\(\Delta S\) 可能在 null 附近跳動。保留固定 pass budget，並對
  extracted frequency、J、\(\kappa\) 做 mesh / pass / sweep-resolution comparison；
* 檢查 power balance、port impedance、S12/S21 reciprocity（在適用時）；
* 若 full-chip 結果超出 RAM，先驗證單 channel 或 channel pair。不要把 coarse full-chip result
  偽裝成已收斂的 final value。

## 6. 流程 C：Flux-line 到 loop 的 mutual inductance

這是本 repo 最容易因 port 設錯而得到錯結果的 HFSS 流程。

### 6.1 物理量與正確公式

在 loop port 開路的條件下：

```text
V_loop = j ω M I_flux
M = Im(Z21) / (2π f)
I_Phi0 = Phi0 / |M|,  Phi0 = 2.068e-15 Wb
```

目標通常是約 0.1–1 pH；太小需要過大 bias current，太大可能帶來 flux-noise coupling。

### 6.2 建立兩埠模型

1. **Port 1：flux-line port。** 放在 flux-line 的 pin / launch 端，跨 CPW signal 與 ground，50 Ω。
2. **Port 2：JJ gap 上的 loop port。** 這張 lumped port sheet 只量測 loop 開路 EMF。
3. **Port 2 必須是 `port only`。** 不能在 port 2 並聯 \(L_J\)。若把 junction inductance 並聯，loop
   會經 \(L_J+L_{array}\) 關閉，得到的 M 會被 \(L_J/(L_J+L_{array})\) 壓低。
4. fluxonium 使用與 layout 相符的 `FluxoniumArrayJJ` simulation model：短的 central JJ gap、兩條
   lumped-L array side、金屬 closing end；不可用一條直線 inductor 代替。
5. readout / charge line 的端點設定為 open termination；flux line return path 與 ground plane 保持連續。
6. 使用 local model 時記錄它省略了哪些全晶片 routing；M 對真實 return path 可能改變數 % 到數十 %。

### 6.3 HFSS setup

1. Solver: Driven Modal / impedance extraction，adaptive frequency 約 100 MHz。
2. 使用 discrete sweep：10–100 MHz、約 10 points。不要用 interpolating sweep 當 reference。
3. 常用嚴格設定是：max \(\Delta S\) 0.0002、至少 12 pass、最多約 15 pass、至少 3 consecutive
   converged passes；但應以目標 Design 的設定為準。
4. 對 flux line、SQUID / junction gap、loop / array 導體加入足夠細的 length mesh。低頻 pH signal 對
   關鍵局部 mesh 很敏感。
5. 保存整個 Z matrix，而非只輸出 Z21。

### 6.4 後處理與 sanity checks

每個 sweep point 計算：

```text
M(f)        = Im(Z21) / (2πf)
L_flux(f)   = Im(Z11) / (2πf)
L_loop(f)   = Im(Z22) / (2πf)
```

HFSS excitation list 和輸出的 Z-matrix index 可能順序不同。辨認 JJ / loop port 的可靠方式是：它應具有
遠大於 flux-line port 的 \(\operatorname{Im}Z_{ii}/\omega\)（loop 通常是 nH，短 flux U-loop 通常是 pH）。

必做檢查：

* \(Z_{21}=Z_{12}\) 的相對誤差很小；
* \(\operatorname{Re}Z_{21}\approx0\)；
* \(M(f)\) 在 10–100 MHz 接近平坦；若隨 \(f^2\) 上升，可能已有 capacitive pickup；
* \(L_{loop}\) 與 layout 設定的 array / JJ inductance 加幾何電感相符；
* 比較兩個 pass budgets 的 M；12 到 15 pass 的差應小於約 1%；
* 用 Biot–Savart filament estimate 作獨立幾何檢查。HFSS 因 flux focusing 比自由空間結果高一些合理，
  但相反方向或差到兩倍以上通常代表模型 / port 有問題。

### 6.5 多 qubit crosstalk matrix

對每條 flux line 作激發、對每個 loop 放 port-only junction gap，建立 \(M_{ij}\)：line \(j\) 對 loop \(i\)。
報告時使用：

```text
crosstalk(i <- j) = M_ij / M_jj
```

還要輸出從低頻到較高頻的 \(M_{ii}(f)/M_{ii}(f_0)\) 與相位，因為這代表 flux pulse 的 on-chip
transfer function，而不只是靜態 bias efficiency。

## 7. 流程 D：Linear qubit / tunable-coupler Eigenmode sweep

這一流程用於檢查 Q1–TC–Q2 等線性化核心，而不是直接宣稱 qubit nonlinearity。

1. 只渲染需要的 qubit islands、coupler 和 readout claws / stubs；不需要的長控制線不要帶入。
2. 以 layout 的 per-junction \(L_J\) 變數建立 linear junction sheets。非對稱 SQUID 的每個 branch 都要有
   對應的 variable；兩個 branch 的並聯關係由 geometry / circuit 反映。
3. 設定 3 個或更多 eigenmodes，最低頻率低於預期 qubit / coupler linear modes。
4. 在數個 coupler \(L_J\) 點重跑同一 geometry。每點都保存 mode list、convergence table、實際 \(L_J\)。
5. 用 linear circuit model 對跨 \(L_J\) 的 mode frequency 做 fit；若 direct qubit–qubit coupling 不受
   mode sweep 約束，使用 Q3D \(C^{-1}\) 作明確標示的 prior。
6. 從 fitted \(C^{-1}\) 計算 \(E_C\) 與 `g_ij n_i n_j` charge-unit coupling。不要混為 exchange coupling
   \(g_{ex}\)，更不要把線性 HFSS mode frequency 寫成 qubit \(f_{01}\)。

## 8. 流程 E：Qubit-readout Eigenmode + pyEPR（選用）

此流程在 native HFSS Eigenmode design 已正確設定後才做。它適合取得 linear-mode basis 上的 EPR
quantity，作為 qubit-readout interaction 的 cross-check。

1. 先完成含 qubit、讀出 channel、feedline port、ground straps、10 µm mesh 的 eigenmode solve。
2. 對每個 JJ 在 pyEPR 中登錄：junction name、對應的 \(L_J\) variable、junction rectangle、junction line、
   (C_J\) variable。
3. 執行 junction information validation；若 rectangle / line 方向或物件名錯，停止，不要使用結果。
4. 設定 dielectric participation 的 material region，執行 EPR analysis。
5. 執行 quantum analysis，並記錄 cosine truncation、Fock truncation、所有 variations。
6. 保存 mode frequency、anharmonicity \(\alpha\)、cross-Kerr matrix \(\chi\)、Lamb shift、mode linewidth
   與由 qubit-like mode linewidth 換算的 Purcell-limited \(T_1\)。

EPR 結果也有模型前提：lumped linear junction、PEC、chosen dielectric loss model、truncation。它不是量測值，
也不代表包含 \(T_1/T_2\)、flux noise 或 readout-chain noise 的實際 device performance。

## 9. Purcell readout 的完整手動順序

若目標是 `feedline – C_p – lambda/4 filter – hammer C_j – lambda/4 resonator – qubit`，建議嚴格依序：

1. **Q3D coupons：** 取得 hammer \(C_j\)、head excess、\(C_p\) 的 mutual / ground load，以及含 qubit
   的 resonator end load。
2. **Circuit model：** 用 `code/resonator/purcell_readout_model.py` 的 `length_for_f` 算 filter / resonator
   filleted lengths；確認 `length_report` 與 target 一致。
3. **HFSS eigenmode：** 用相同幾何確認兩個 mode、初步 \(f_p,f_r,J,\kappa_p\)。
4. **HFSS driven S21：** 用 wide + fine sweep、complex-pole fit 決定最終 \(J\) 與 \(\kappa_p\)。
5. **若頻率需 trim：** 只改短路臂長度，使用 \(\Delta L=\Delta f/s\)。不要改 hammer 或 tap；否則會同時
   改變 J。
6. **Loaded check：** 加入 qubit linear junction，檢查 \(\chi\)、\(\kappa_r\) 與 mode separation。
7. **若有 series \(C_{in}\)：** 建立包含所有 tee、channel 與 feedline ports 的 full-feed case。單一 channel
   的 \(\kappa_p\) 不能代表 standing-wave environment。

對 `PurcellReadout`，還必須驗證非-hammer mutual 小於 \(0.1 C_j\)。若做不到，先處理平行 CPW run 或
ground-return coupling，而非直接把差異當成 fitted J。

## 10. 常見失敗與排錯順序

| 症狀 | 優先檢查 |
|---|---|
| 沒有預期 S21 dip | wide sweep resolution、open end 是否被短路、port sheet、mode 是否遠失諧 |
| 頻率錯很多 | geometry revision、filleted length、open/short、Q3D end load、單位、junction L |
| Purcell J 遠大於 hammer model | ground straps、parallel CPW、hammer pocket ground return、未預期的長平行段 |
| M 太小很多 | JJ port 是否誤與 \(L_J\) 並聯、loop port index 是否辨認錯、mesh、return path |
| M 不隨頻率平坦 | 太接近 resonance、capacitive pickup、interpolating sweep artifact、port / net setup |
| HFSS Analyze COM error | scratch disk、OOM、斷開 net、AEDT message window；不要先猜 Python 問題 |
| AEDT 沒有使用 mesh link | 檢查 `ImportMesh` 狀態，不要只因 menu 操作成功就假定 mesh 已匯入 |
| 結果在不同 rerun 差很大 | object selection、mesh coverage、wire bonds / straps、solver pass、geometry / input snapshot |

## 11. 每次手動 run 的提交前 checklist

- [ ] Design、geometry revision、simulation components 和上游 run 已寫下。
- [ ] 未以 fab GDS 或手工重畫幾何代替 simulation geometry。
- [ ] Ports 的位置、方向、50 Ω reference、open / short terminations 已逐項檢查。
- [ ] Junction boundary / lumped L 是 layout input，已記錄而未誤稱為 HFSS output。
- [ ] 所有 CPW / capacitor / qubit metal 有正確 length-based mesh；driven case 有 explicit mesh。
- [ ] Wire bonds 與短 stub 的 PEC straps 都已檢查接地連續。
- [ ] 已匯出 complex raw S / Z / eigenfrequency，以及 convergence table。
- [ ] 結果的單位清楚：\(f\) vs. \(\omega\)，\(\kappa\) vs. \(\kappa/2\pi\)，charge coupling vs. exchange coupling。
- [ ] 與前一版或較嚴格 mesh / pass 的差異已評估；不足時標為 `unknown` 或 `needs_revalidation`。
- [ ] 新 run 沒有覆寫歷史 raw data，且 `design.json` 不會自動改採最新結果。

## 12. 對照本 repo 的正式規範與腳本

這份學習筆記的詳細規則來源仍是：

* `skill/em_simulation.md`：環境、default pass、mesh、wire bond、HFSS practical lessons；
* `skill/mutual_inductance_analysis.md`：M 的 two-port setup、驗證與 reporting；
* `skill/purcell_resonator_design.md` 與 `note/purcell_filter_method.md`：Purcell channel 的物理與驗證順序；
* `skill/parameter_provenance.md`：run storage、validation 與 final adoption；
* 各 Design 的 `simulation/resonator/`、`simulation/flux/`、`simulation/qubit/` 下的既有 HFSS entry point：
  它們才定義該 chip 的 component list、port names、頻率範圍、geometry override 與已知限制。

若學習目標是第一個可重現的操作，建議先選一個單 channel Purcell module 的 eigenmode + driven S21；
確認能從同一 native geometry 得到兩個模式和 complex S21 後，再進到 full-feed readout 或 low-frequency
mutual inductance。後兩者對 port / return path 的設定更敏感。
