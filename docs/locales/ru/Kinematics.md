# Kinematics

В этом документе представлен обзор того, как Klipper реализует движение робота (его [кинематика] (https://en.wikipedia.org/wiki/Kinematics )). Содержание может представлять интерес как для разработчиков, заинтересованных в работе над программным обеспечением Klipper, так и для пользователей, заинтересованных в лучшем понимании механики своих машин.

## Ускорение

Klipper реализует схему постоянного ускорения всякий раз, когда печатающая головка меняет скорость - скорость постепенно изменяется на новую скорость, а не внезапно переключается на нее. Клиппер всегда обеспечивает ускорение между инструментальной головкой и отпечатком. Нить, выходящая из экструдера, может быть довольно хрупкой - быстрые рывки и / или изменение потока экструдера приводят к низкому качеству и плохой адгезии к слою. Даже при отсутствии выдавливания, если печатающая головка находится на том же уровне, что и печать, быстрое подергивание головки может привести к разрыву недавно нанесенной нити накала. Ограничение изменения скорости печатающей головки (относительно печати) снижает риск прерывания печати.

Также важно ограничить ускорение, чтобы шаговые двигатели не пропускали и не создавали чрезмерной нагрузки на машину. Klipper ограничивает крутящий момент на каждом шаговом механизме за счет ограничения ускорения печатающей головки. Принудительное ускорение печатающей головки, естественно, также ограничивает крутящий момент шаговых двигателей, которые перемещают печатающую головку (обратное не всегда верно).

Klipper реализует постоянное ускорение. Ключевой формулой для постоянного ускорения является:

```
velocity(time) = start_velocity + accel*time
```

## Генератор трапеций

В Klipper используется традиционный "генератор трапеций" для моделирования движения каждого хода - каждый ход имеет начальную скорость, разгоняется до крейсерской скорости с постоянным ускорением, движется с постоянной скоростью, а затем замедляется до конечной скорости с постоянным ускорением.

![trapezoid](img/trapezoid.svg.png)

Его называют "генератором трапеций", потому что диаграмма скорости движения похожа на трапецию.

Крейсерская скорость всегда больше или равна как начальной, так и конечной скорости. Фаза разгона может быть нулевой (если начальная скорость равна крейсерской), фаза крейсерской может быть нулевой (если после разгона движение сразу начинает замедляться), и/или фаза замедления может быть нулевой (если конечная скорость равна крейсерской).

![trapezoids](img/trapezoids.svg.png)

## Look-ahead

Система "look-ahead" используется для определения скорости прохождения поворотов между ходами.

Рассмотрим следующие два движения, расположенные на плоскости XY:

![corner](img/corner.svg.png)

В описанной выше ситуации можно полностью замедлиться после первого движения и затем полностью ускориться в начале следующего движения, но это не идеальный вариант, поскольку все эти ускорения и замедления значительно увеличат время печати, а частые изменения потока в экструдере приведут к ухудшению качества печати.

Чтобы решить эту проблему, механизм "look-ahead" ставит в очередь несколько входящих ходов и анализирует углы между ними, чтобы определить разумную скорость, которую можно получить на "стыке" между двумя ходами. Если следующий ход почти в том же направлении, то голове нужно лишь немного замедлиться (если это вообще возможно).

![lookahead](img/lookahead.svg.png)

Однако если следующий ход образует острый угол (голова будет двигаться почти в обратном направлении на следующем ходу), то допускается лишь небольшая скорость перехода.

![lookahead](img/lookahead-slow.svg.png)

Скорости на перекрестках определяются с помощью "приближенного центростремительного ускорения". Лучшее [описано автором](https://onehossshay.wordpress.com/2011/09/24/improving_grbl_cornering_algorithm/). Однако в Klipper скорости переходов настраиваются путем указания желаемой скорости, которую должен иметь угол 90° ("квадратная угловая скорость"), а скорости переходов для других углов определяются исходя из этого.

Key formula for look-ahead:

```
end_velocity^2 = start_velocity^2 + 2*accel*move_distance
```

### Minimum cruise ratio

Klipper also implements a mechanism for smoothing out the motions of short "zigzag" moves. Consider the following moves:

![zigzag](img/zigzag.svg.png)

In the above, the frequent changes from acceleration to deceleration can cause the machine to vibrate which causes stress on the machine and increases the noise. Klipper implements a mechanism to ensure there is always some movement at a cruising speed between acceleration and deceleration. This is done by reducing the top speed of some moves (or sequence of moves) to ensure there is a minimum distance traveled at cruising speed relative to the distance traveled during acceleration and deceleration.

Klipper implements this feature by tracking both a regular move acceleration as well as a virtual "acceleration to deceleration" rate:

![smoothed](img/smoothed.svg.png)

Specifically, the code calculates what the velocity of each move would be if it were limited to this virtual "acceleration to deceleration" rate. In the above picture the dashed gray lines represent this virtual acceleration rate for the first move. If a move can not reach its full cruising speed using this virtual acceleration rate then its top speed is reduced to the maximum speed it could obtain at this virtual acceleration rate.

For most moves the limit will be at or above the move's existing limits and no change in behavior is induced. For short zigzag moves, however, this limit reduces the top speed. Note that it does not change the actual acceleration within the move - the move continues to use the normal acceleration scheme up to its adjusted top-speed.

## Generating steps

Once the look-ahead process completes, the print head movement for the given move is fully known (time, start position, end position, velocity at each point) and it is possible to generate the step times for the move. This process is done within "kinematic classes" in the Klipper code. Outside of these kinematic classes, everything is tracked in millimeters, seconds, and in cartesian coordinate space. It's the task of the kinematic classes to convert from this generic coordinate system to the hardware specifics of the particular printer.

Klipper uses an [iterative solver](https://en.wikipedia.org/wiki/Root-finding_algorithm) to generate the step times for each stepper. The code contains the formulas to calculate the ideal cartesian coordinates of the head at each moment in time, and it has the kinematic formulas to calculate the ideal stepper positions based on those cartesian coordinates. With these formulas, Klipper can determine the ideal time that the stepper should be at each step position. The given steps are then scheduled at these calculated times.

The key formula to determine how far a move should travel under constant acceleration is:

```
move_distance = (start_velocity + .5 * accel * move_time) * move_time
```

and the key formula for movement with constant velocity is:

```
move_distance = cruise_velocity * move_time
```

The key formulas for determining the cartesian coordinate of a move given a move distance is:

```
cartesian_x_position = start_x + move_distance * total_x_movement / total_movement
cartesian_y_position = start_y + move_distance * total_y_movement / total_movement
cartesian_z_position = start_z + move_distance * total_z_movement / total_movement
```

### Cartesian Robots

Generating steps for cartesian printers is the simplest case. The movement on each axis is directly related to the movement in cartesian space.

Key formulas:

```
stepper_x_position = cartesian_x_position
stepper_y_position = cartesian_y_position
stepper_z_position = cartesian_z_position
```

### CoreXY Robots

Generating steps on a CoreXY machine is only a little more complex than basic cartesian robots. The key formulas are:

```
stepper_a_position = cartesian_x_position + cartesian_y_position
stepper_b_position = cartesian_x_position - cartesian_y_position
stepper_z_position = cartesian_z_position
```

### Delta Robots

Step generation on a delta robot is based on Pythagoras's theorem:

```
stepper_position = (sqrt(arm_length^2
                         - (cartesian_x_position - tower_x_position)^2
                         - (cartesian_y_position - tower_y_position)^2)
                    + cartesian_z_position)
```

### Stepper motor acceleration limits

With delta kinematics it is possible for a move that is accelerating in cartesian space to require an acceleration on a particular stepper motor greater than the move's acceleration. This can occur when a stepper arm is more horizontal than vertical and the line of movement passes near that stepper's tower. Although these moves could require a stepper motor acceleration greater than the printer's maximum configured move acceleration, the effective mass moved by that stepper would be smaller. Thus the higher stepper acceleration does not result in significantly higher stepper torque and it is therefore considered harmless.

However, to avoid extreme cases, Klipper enforces a maximum ceiling on stepper acceleration of three times the printer's configured maximum move acceleration. (Similarly, the maximum velocity of the stepper is limited to three times the maximum move velocity.) In order to enforce this limit, moves at the extreme edge of the build envelope (where a stepper arm may be nearly horizontal) will have a lower maximum acceleration and velocity.

### Extruder kinematics

Klipper implements extruder motion in its own kinematic class. Since the timing and speed of each print head movement is fully known for each move, it's possible to calculate the step times for the extruder independently from the step time calculations of the print head movement.

Basic extruder movement is simple to calculate. The step time generation uses the same formulas that cartesian robots use:

```
stepper_position = requested_e_position
```

### Повышение давления

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

Key formula for "smoothed pressure advance":

```
smooth_pa_position(t) =
    ( definitive_integral(pa_position(x) * (smooth_time/2 - abs(t - x)) * dx,
                          from=t-smooth_time/2, to=t+smooth_time/2)
     / (smooth_time/2)^2 )
```
