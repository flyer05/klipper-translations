# 複數微控制器歸零與探高

Klipper支援歸零限位開關和動作的步進電機連線到不同的微控制器上。該功能被稱為「複數微控制器歸零」。該功能也支援將探針連線到不同的微控制器上。

該功能可以簡化接線，因為限位開關或探針可以連線到距離最短的微控制器上。然而，該功能也會帶來問題，因為運動控制器和限位控制器並非同一控制器，可能造成歸零或探高時的「過度運動」。

過度運動的可能成因是，控制步進電機運動的微控制器 和 監控限位開關的微控制器之間的資訊傳遞存在延時。Klipper在設計上將延時壓縮到25ms以下。（在使用複數微控制器時，各個微控制器會通過週期性發送狀態資訊確定與上位機的延時不超過25ms。）

例如，如果歸零速度為10 mm/s則可能的過運動的量為0.25mm（10mm/s * .025s == 0.250mm）。在進行復數微控制器的歸零配置時應充分考慮過運動的影響。使用低速歸零可以有效減少過運動。

步進電機的過運動不太可能對歸零和探高的精度產生很大的影響。Klippe程式碼上會考慮通訊延時校正歸零的結果。但是，過運動對硬體穩固性有要求，因為過運動發生時有可能會損壞硬體。

In order to use this "multi-mcu homing" capability the hardware must have predictably low latency between the host computer and all of the micro-controllers. Typically the round-trip time must be consistently less than 10ms. High latency (even for short periods) is likely to result in homing failures.

Should high latency result in a failure (or if some other communication issue is detected) then Klipper will raise a "Communication timeout during homing" error.

要注意，當一個軸由多個步進電機控制（如`stepper_z`和`stepper_z1`），這些電機必須連線到同一微控制器上以實現複數微控制器歸零。詳細來說，即Z限位開關位於微控制器1， `stepper_z`連線到微控制器2，則`stepper_z1`必須連線到微控制器2。
