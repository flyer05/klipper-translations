# 探針校準

本文件介紹了在 Klipper 中校準"自動 Z 探針" X、Y 和 Z 偏移的流程。這對於配置檔案中具有 `[probe]` 或 `[bltouch]` 部分的印表機非常有用。

## 探針X/Y偏移校準

為校正X、Y偏移值，首先打開OctoPrint中的「控制（control）」子頁面，進行三軸的歸零，然後用OctoPrint內的手調按鈕將噴嘴移動到熱床的中央位置。

探針下方的熱床上貼上一片美紋紙（或類似的薄片）。轉到OctoPrint的「命令列（Terminal）」子頁，輸入PROBE 命令並回車：

```
PROBE
```

在探針正下方的美紋紙上，用記號筆標註探針觸發的位置（或使用其他方法記錄下探針觸發時的物理位置）。

在命令列中輸入`GET_POSITION`命令，回車，記錄下此時列印頭的XY位置。輸出如下：

```
Recv: // toolhead: X:46.500000 Y:27.000000 Z:15.000000 E:0.000000
```

此時，我們可以知道探針觸發的X座標為46.5，Y座標為27。

記錄下上述的座標后，在命令列使用一系列的 G1 命令，使噴嘴移動到熱床的記號的正上方。例如：

```
G1 F300 X57 Y30 Z15
```

將噴嘴移動到X 57， Y 30上。當噴嘴的位置剛好位於幾號上時，在命令列鍵入 `GET_POSITION` 獲得此時噴嘴所在的座標。

X偏移值為 `噴嘴X座標值 - 探針X座標值`， 類似地， Y偏移值為 `噴嘴Y座標值 - 探針Y座標值`。將上述數值更新到printer.cfg檔案內，掀掉熱床上的美紋紙/膠帶，並重啟Klipper以使設定生效。

## 探針Z偏移值校準

準確的探針 z 偏移(z_offset)是高質量列印的基礎。z 偏移是探針觸發時探針和噴嘴之間的高度差。Klipper 中的 `PROBE_CALIBRATE`（探針校準）工具可用於測量這個值——首先，該工具會執行一次自動探測以獲取探針的 z 觸發位置，然後需要手動調整Z座標以獲取噴嘴碰觸到熱床時的 z 高度。然後將根據這些測量值計算探針的 z 偏移。

首先進行三軸的歸零，然後將噴嘴移動到熱床的中央位置。轉到OctoPrint的「命令列（Terminal）」子頁，輸入 `PROBE_CALIBRATE`以啟動z_offset校準工具。

工具首先會令探針進行一次自動探測，獲取觸發探針的z位置，之後，控制噴嘴上升，並將噴嘴的X/Y位置移動到探針對應位置上，並開始手動調平流程。如果噴嘴沒有移動到探針進行自動探測的位置，輸入`ABORT`以停止手動調平，並上文根據X、y偏移校準流程進行探針X、Y校準。

Once the manual probe tool starts, follow the steps described at ["the paper test"](Bed_Level.md#the-paper-test) to determine the actual distance between the nozzle and bed at the given location. Once those steps are complete one can `ACCEPT` the position and save the results to the config file with:

```
SAVE_CONFIG
```

注意！如果修改了印表機的運動系統、噴嘴位置或探針位置中的任意一項，PROBE_CALIBRATE的結果將會需要重新測量。

如果探針的物理X、Y偏移量，或熱床的傾斜度發生變化（如 調整了熱床調平螺母，進行DELTA_CALIBRATE，進行Z_TILT_ADJUST，進行QUAD_GANTRY_LEVEL或其他行為），也應進行一次PROBE_CALiRATE。

上述使PROBE_CALIBRATE結果失效的行為，同樣會使使用探針測量的[床網](Bed_Mesh.md)結果失效。推薦在完成PROBE_CALIBRATE后再進行一次BED_MESH_CALIBRATE。

## 重複性測試

在完成探針的X、Y、Z偏移的校準后，推薦進行探針的重複性測試。首先對印表機進行三週歸零，然後將噴嘴移動到熱床的中央位置。進入OctoPrint的命令列界面，執行`PROBE_ACCURACY`命令。

該命令會就地進行10次探針測量，並輸出類似下方的結果：

```
Recv: // probe accuracy: at X:0.000 Y:0.000 Z:10.000
Recv: // and read 10 times with speed of 5 mm/s
Recv: // probe at -0.003,0.005 is z=2.506948
Recv: // probe at -0.003,0.005 is z=2.519448
Recv: // probe at -0.003,0.005 is z=2.519448
Recv: // probe at -0.003,0.005 is z=2.506948
Recv: // probe at -0.003,0.005 is z=2.519448
Recv: // probe at -0.003,0.005 is z=2.519448
Recv: // probe at -0.003,0.005 is z=2.506948
Recv: // probe at -0.003,0.005 is z=2.506948
Recv: // probe at -0.003,0.005 is z=2.519448
Recv: // probe at -0.003,0.005 is z=2.506948
Recv: // probe accuracy results: maximum 2.519448, minimum 2.506948, range 0.012500, average 2.513198, median 2.513198, standard deviation 0.006250
```

理想狀況下，使用探針測量的所有結果應該一致（也就是10次測量的結果為同一值）。然而，測量結果的最大值和最小值之間差距 「一個z電機步長」或5微米，也是正常的情況。一個「電機步長」是`旋轉一週的長度/(旋轉一週需要的步數*驅動微步設定)`。測量最大值和最小值的差值稱為偏差範圍。故在上述的例子中，因為印表機的z步長為0.0125，因此誤差範圍在0.012500mm可以認為是正常值。

如果測試結果顯示範圍(range)值大於25微米（0.025毫米），那麼探針不滿足典型的床面調平流程的精確度要求。可以嘗試調整探測速度和/或探測起點高度以提高探頭的重複性。`PROBE_ACCURACY`命令允許使用不同的參數進行測試，以瞭解它們的影響 - 請參閱[G-Code文件](G-Code.md#probe_accuracy)瞭解更多細節。如果探針在大多數情況下能獲得可重複的結果，但偶爾會出現異常值，那麼可以增加在每個探測點的探測次數來解決這個問題--詳見[配置參考](Config_Reference.md#probe)中關於探針`samples`配置參數的描述。

要更改探針測試速度，重複採樣或其他設定，應在修改printer.cfg使用`RESTART`命令以應用修改值。推薦在使用新設定值后再進行一次[Z偏移校準](#calibrating-probe-z-offset)。如果重複性測試結論不能接受，建議不要使用自動熱床調平功能。Klipper提供了數種手動調平的工具，詳情請見[列印床調平](Bed_Level.md)。

## 區域性偏差確定

一些探針可能具有與位置相關的系統性偏差。比如，由於探針安裝失誤，探針沿Y軸移動會產生傾斜，那麼探針在沿Y軸進行測量得出的結果會存在偏差。

上述狀況往往出現在三角洲印表機上，然而，所有印表機均有可能發生上述狀況。

我們可以在不同的XY位置上使用`PROBE_CALIBRATE`命令測量z_offset來確定位置偏差。理想情況下，z_offset在任意位置均為同一讀值。

對於三角洲印表機，請嘗試依次在靠近A柱、B柱和C柱的位置測量z_offset。對於龍門、corexy或類似結構印表機，嘗試在熱床的四個角進行z_offset的測量。

在進行上述測試前，應先按照本文件的開頭部分對X、Y、Z的偏移值進行校準，然後對印表機進行歸零，並將探針移動到首個目標XY位置。按照[探針Z偏移值校準](#探針Z偏移值校準)的步驟，執行`PROBE_CALIBRATE`，重複多次`TESTZ`命令，並在噴嘴觸床時使用 `ACCEPT`命令，但是**切勿使用 `SAVE_CONFIG`**。手動記下此時的z_offset 。之後將探針移動到其他XY位置，重複上述的`PROBE_CALIBRATE`步驟，並分別記下z_offset。

如果上述方法中測出的最大z_offset 和最小z_offset 之間的差值大於25微米（.025mm），則該探針不適用于常規的熱床調平。此時應參照 [熱床調平](Bed_Level.md) 文件的手動調平部分進行調平。

## 溫度偏差

對於多種形式的探針，在不同的溫度下工作均具有一定的系統性偏差。比如，在高溫下，探針可能總是在更低的高度下觸發。

針對這種偏差，建議在恒定的溫度下進行熱床調平。即，要麼總是在室溫下進行床網測量，要麼總要在工作溫度下進行。無論採取哪種方案，都推薦在達到目標溫度數分鐘后再進行測量，以便印表機始終處於目標溫度。

要測量溫度偏差，首先確保印表機處於室溫，對三軸進行歸零，然後將列印頭移動到熱床中央，執行`PROBE_ACCURACY`命令。記下此時的讀數。之後，在不歸零或關閉電機的情況下，加熱噴嘴和熱床到工作溫度，並再次執行`PROBE_ACCURACY`。理想情況下，兩次探針測量會得出相同的結果。但若溫度偏差存在，建議每次達到工作溫度后再進行測量。
