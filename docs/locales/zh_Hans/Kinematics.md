# 运动学

该文档提供Klipper实现机械[运动学](https://en.wikipedia.org/wiki/Kinematics)控制的概述，以供 致力于完善Klipper的开发者 或 希望对自己的设备的机械原理有进一步了解的爱好者 参考。

## 加速

Klipper总使用常加速度策略——打印头的速度总是梯度变化到新的速度，而非使用速度突变的方式。Klipper着眼于打印件和打印头之间的速度变化。离开挤出机的耗材十分脆弱，突然的移动速度和/或挤出流量突变可能会导致造成打印质量或床黏着能力的下降。甚至在无挤出时，如果打印头和打印件顶端在同一水平面时，喷嘴的速度突变有可能对刚挤出的耗材进行剐蹭。限制打印头相对于打印件的速度，可以减少剐蹭打印件的风险。

限制减速度也能减少步进电机丢步和炸机的状况。Klipper通过限制打印头的加速度来限制每个步进电机的力矩。限制打印头的加速度自然也限制了移动打印头的步进器的扭矩（反之则不一定）。

Klipper实现恒加速度控制，关键的方程如下：

```
velocity(time) = start_velocity + accel*time
```

## 梯形发生器

Klipper 使用传统的"梯形发生器"来产生每个动作的运动--每个动作都有一个起始速度，先恒定的加速度加速到一个巡航速度，再以恒定的速度巡航，最后用恒定的加速度减速到终点速度。

![trapezoid](img/trapezoid.svg.png)

因为移动时的速度图看起来像一个梯形，它被称为 "梯形发生器"。

巡航速度总是大于等于起始和终端速度。加速度阶段可能持续时间为0（如果起始速度等于巡航速度），巡航速度的持续时间也可为0（如果在加速结束后马上进行减速），减速阶段也能为0（如果终端速度等于巡航速度）。

![trapezoids](img/trapezoids.svg.png)

## 预计算（look-ahead）

拐角速度使用预计算系统进行处理。

考虑以下两个在 XY 平面上的移动：

![corner](img/corner.svg.png)

在上述的状况下，打印机可以在第一步时减速至停止，并在第二步开始时加速至巡航速度。但这种运动策略并不理想，完全减速和完全加速会大幅增加打印时间，同时挤出量会频繁变动，从而导致打印质量的下降。

要解决这种情况，klipper引入了预计算机制，预先依次计算后续的数个移动，分析其中的拐角并确定合适的拐角速度。如果下一步的速度与现时的移动速度相近，则滑车速度仅会稍微减少。

![lookahead](img/lookahead.svg.png)

然而，如果下一步形成一个尖锐的拐角（滑车将在下一步进行近于反方向的移动），则只能采用一个很低的拐角速度。

![lookahead](img/lookahead-slow.svg.png)

转角速度由“近似向心加速度”确定。最好[由作者描述](https://onehossshay.wordpress.com/2011/09/24/improving_grbl_cornering_algorithm/)。然而在Klipper中转角速度是通过指定90°角应该有的理想速度（“直角速度”）并且其他的角度的转角速度也是根据它推导出来的。

预计算的关键方程：

```
end_velocity^2 = start_velocity^2 + 2*accel*move_distance
```

### 预计算结果平滑

Klipper 实现了一种用于平滑短距离之字形移动的机制。参考以下移动：

![zigzag](img/zigzag.svg.png)

在上述情况下，从加速到减速的频繁变化会导致机器振动并且会对机器造成压力和加噪音。为了减少这种情况，Klipper既跟踪常规的移动加速度并且也跟踪虚拟的"加减速率"。利用这个系统，这些短的"zigzag"移动的最高速度被限制以使得打印机的运动可以更加平滑：

![smoothed](img/smoothed.svg.png)

具体来说，代码计算的是限制在这个虚拟的“加速到减速”率下时（默认为正常加速率的一半），每个动作的速度是多少。在上图中，灰色虚线代表了第一段移动时的虚拟加速率。如果一段移动使用这个虚拟加速度不能达到目标巡航速度，那么这段移动的最高速度将被降低到它在这个虚拟加速率下所能获得的最大速度。对于大多数移动来说，该限制将处于或高于该移动的现有限制，并且不会改变移动的行为。然而，对于短的 "之 "字形移动，这个限制会降低最高速度。请注意，它不会改变移动中的实际加速度--移动会继续使用正常的加速，直到其调整后的最高速度。

## 生成步数（Generating steps）

前瞻过程完成后给定移动的打印头运动已被确定（时间、开始位置、结束位置、每一点的速度），可以被用于生成移动的步进时间。这个过程是在Klipper代码的运动学类中完成的。在这些运动学类之外，所有的东西都是以毫米、秒为单位，在笛卡尔坐标空间进行跟踪。运动学类负责将这个通用坐标系统转换为符合打印机硬件特性的坐标系。

Klipper使用一个[迭代求解器](https://zh.wikipedia.org/wiki/%E6%B1%82%E6%A0%B9%E7%AE%97%E6%B3%95)来生成每个步进的步进时间。该代码包含了计算打印头在每个时间点上的理想笛卡尔坐标的公式，它还有运动学公式来计算基于这些笛卡尔坐标的理想步进位置。通过这些公式，Klipper可以确定步进电机在每个步进位置时的理想步进时间。然后在这些计算出的时间内安排给定的步进。

确定一个移动在恒定加速度下应该移动多远的关键公式是：

```
move_distance = (start_velocity + .5 * accel * move_time) * move_time
```

并且匀速运动的关键公式是：

```
move_distance = cruise_velocity * move_time
```

在给定移动距离的情况下用于确定移动的笛卡尔坐标的关键公式为：

```
cartesian_x_position = start_x + move_distance * total_x_movement / total_movement
cartesian_y_position = start_y + move_distance * total_y_movement / total_movement
cartesian_z_position = start_z + move_distance * total_z_movement / total_movement
```

### 笛卡尔机器

为笛卡尔坐标的打印机生成步进是最简单的情况。每个轴上的运动与笛卡尔空间中的运动直接相关。

关键公式：

```
stepper_x_position = cartesian_x_position
stepper_y_position = cartesian_y_position
stepper_z_position = cartesian_z_position
```

### CoreXY 机器人

在CoreXY的机器上生成步进只比基本的卡特尔机器人复杂一点。关键公式是：

```
stepper_a_position = cartesian_x_position + cartesian_y_position
stepper_b_position = cartesian_x_position - cartesian_y_position
stepper_z_position = cartesian_z_position
```

### 三角洲机器

三角洲结构机器人上的步进生成基于勾股定理：

```
stepper_position = (sqrt(arm_length^2
                         - (cartesian_x_position - tower_x_position)^2
                         - (cartesian_y_position - tower_y_position)^2)
                    + cartesian_z_position)
```

### 步进电机加速限制

With delta kinematics it is possible for a move that is accelerating in cartesian space to require an acceleration on a particular stepper motor greater than the move's acceleration. This can occur when a stepper arm is more horizontal than vertical and the line of movement passes near that stepper's tower. Although these moves could require a stepper motor acceleration greater than the printer's maximum configured move acceleration, the effective mass moved by that stepper would be smaller. Thus the higher stepper acceleration does not result in significantly higher stepper torque and it is therefore considered harmless.

However, to avoid extreme cases, Klipper enforces a maximum ceiling on stepper acceleration of three times the printer's configured maximum move acceleration. (Similarly, the maximum velocity of the stepper is limited to three times the maximum move velocity.) In order to enforce this limit, moves at the extreme edge of the build envelope (where a stepper arm may be nearly horizontal) will have a lower maximum acceleration and velocity.

### Extruder kinematics

Klipper implements extruder motion in its own kinematic class. Since the timing and speed of each print head movement is fully known for each move, it's possible to calculate the step times for the extruder independently from the step time calculations of the print head movement.

Basic extruder movement is simple to calculate. The step time generation uses the same formulas that cartesian robots use:

```
stepper_position = requested_e_position
```

### 压力提前

Experimentation has shown that it's possible to improve the modeling of the extruder beyond the basic extruder formula. In the ideal case, as an extrusion move progresses, the same volume of filament should be deposited at each point along the move and there should be no volume extruded after the move. Unfortunately, it's common to find that the basic extrusion formulas cause too little filament to exit the extruder at the start of extrusion moves and for excess filament to extrude after extrusion ends. This is often referred to as "ooze".

![ooze](img/ooze.svg.png)

The "pressure advance" system attempts to account for this by using a different model for the extruder. Instead of naively believing that each mm^3 of filament fed into the extruder will result in that amount of mm^3 immediately exiting the extruder, it uses a model based on pressure. Pressure increases when filament is pushed into the extruder (as in [Hooke's law](https://en.wikipedia.org/wiki/Hooke%27s_law)) and the pressure necessary to extrude is dominated by the flow rate through the nozzle orifice (as in [Poiseuille's law](https://en.wikipedia.org/wiki/Poiseuille_law)). The key idea is that the relationship between filament, pressure, and flow rate can be modeled using a linear coefficient:

```
pa_position = nominal_position + pressure_advance_coefficient * nominal_velocity
```

See the [pressure advance](Pressure_Advance.md) document for information on how to find this pressure advance coefficient.

The basic pressure advance formula can cause the extruder motor to make sudden velocity changes. Klipper implements "smoothing" of the extruder movement to avoid this.

![pressure-advance](img/pressure-velocity.png)

The above graph shows an example of two extrusion moves with a non-zero cornering velocity between them. Note that the pressure advance system causes additional filament to be pushed into the extruder during acceleration. The higher the desired filament flow rate, the more filament must be pushed in during acceleration to account for pressure. During head deceleration the extra filament is retracted (the extruder will have a negative velocity).

The "smoothing" is implemented using a weighted average of the extruder position over a small time period (as specified by the `pressure_advance_smooth_time` config parameter). This averaging can span multiple g-code moves. Note how the extruder motor will start moving prior to the nominal start of the first extrusion move and will continue to move after the nominal end of the last extrusion move.

"平滑压力提前"的关键公式：

```
smooth_pa_position(t) =
    ( definitive_integral(pa_position(x) * (smooth_time/2 - abs(t - x)) * dx,
                          from=t-smooth_time/2, to=t+smooth_time/2)
     / (smooth_time/2)^2 )
```