# Miscelaneous topics explained

[TOC]

---

## Solver / Numeric Integration

#### Euler or Runge-Kutta 4 (RK4)?

RK4 adds a kind smoothing to the calculations, so rapid changes in the integrated data
are smoothed in some amount. On the other hand, Euler allow fast changing values, provide
immediate response, and the number of substeps can be raised for better accuracy. RK4 always
takes 4 integration substeps

Personally I prefer to use Euler with 4 substeps rather than RK4.

#### Euler substeps

**Substeps** are subdivisions of the Unity physics time steps. Euler can use from 1 to any
number of substeps. Runge-Kutta 4 (RK4) takes 4 substeps always.

The more substeps, the more precise are the results but CPU usage is increased. However I've noticed
no significant penalty on using 1 to 8 substeps. CPU usage as shown at the profiler gets increased
from 10 substeps and above.

Some subjective recommendations:

Substeps  | Comments
:--------:| ---------------
1 |	Non-player vehicles that don't require precise physics such traffic, parked cars, etc.
2 | Minimum recommended for any player vehicle or "active" non-player vehicle (i.e. opponents) that require precise physic behavior.
4 | Good value for having reasonably precise results.
8 | Recommended value for having good precise results. Profiler tests show no significant penalty on using from 1 to 8 substeps.
20 | Maximum recommended value. At Unity's default physic step (0.02s, 50Hz) the vehicle does its internal calculations at **1000 Hz** (20 x 50hz).

Reaching 40 or more substeps is typically not necessary nor recommended. Some components exhibit
numerical oscillations on high amount of substeps.

Final value depends on the specific project: CPU requirements, number and type of vehicles, expected
precision...

#### Should I change the Unity's physics timestep?

There's no need for changing Unity's default physics/fixed timestep (0.02, 50Hhz). Within Vehicle
Physics Pro you can configure the number of integration substeps in a per-vehicle basis.

Some calculations and data are extracted out of the Unity physics engine: _velocity_ and
_downforce_. These are calculated at the Unity's physics rate. Tire forces are also applied at
this rate.

If you really need to get the most precision out of the vehicle physics then you can modify the
Unity's fixed timestep to 0.01 (100Hz) or 0.005 (200Hz). This is NOT recommended because it can
dramatically increase the CPU usage footprint because of the physics. Also ensure the
integration substeps are adjusted accordingly: if fixed timestep is 0.01 (100Hz) and your
vehicle is set to 10 substeps, then the calculations for that vehicle are done at 1000Hz.

#### How to measure the precision of the integration?

Ensure you understand the [difference among _accuracy_ (or _trueness_) and _precision_](http://en.wikipedia.org/wiki/Accuracy_and_precision).
**Vehicle Physics Pro is fully accurate in its design, implementation and behavior** giving
reasonably precise results. _Precision_ affects the the specific numeric values only, and depends
on the integration method and substeps.

A simple way to measure the precision is setting the **Differential Type** to **Locked**. This
enforces both drive wheels to rotate at the same rate. Then drive around gently and do some
left-right turning while watching the rotation rate of the drive wheels at the telemetry.

When using 1-4 substeps the wheels are "pursuing" each other when changing the steering direction,
but won't likely spin at a the same rate unless steering is kept steady. The more substeps, the
faster they will catch each other's rotation rate. With 10-20 substeps the difference in their
rates will always be within 1-2 rpms only.

#### Vehicle shakes or becomes unstable

If the vehicle becomes unstable at high speeds then either increase the tire relaxation rate or
disable the tire relaxation feature. Low tire relaxation rates at high speeds can enter in resonance
with lateral forces destabilizing the vehicle.

!!! Info "&fa-thumbs-o-up; Pro-Tip"
	Tire relaxation rate can be used to simulate tires targeted to different speeds. Tires with
	low spring rates become difficult to drive at high speeds.

If you observe values that are quickly oscillating at the telemetry in a way that visibly affects
the vehicle (shakes) then either change the integration method or adjust the substeps in the
Euler method:

- A single substep is likely to cause oscillating values, but usually they don't have a noticeable
effect.
- 2-8 substeps are pretty stable in most situations.
- 40 or more substeps are not recommended as the numerical oscillations can get magnified.

---

## Engine

#### It's difficult to stall

Increasing the **Stall bias** setting makes the engine more sensitive to rpms below idle. Check out
the calculated stall rpm value at the inspector, below the engine graph.

The **Idle Control** setting defines how the idle rpms are enforced:

- **Active**: Vehicle's electronic system actively applies as much torque as available for keeping
	the idle rpms.
- **Passive**: Engine just applies a constant torque that compensates the friction at idle rpms.
	Works better with a steep friction curve at idle rpm. Also, requires the throttle to be applied
	for starting the engine with the ignition key.

The Passive mode is much easier to stall, but requires configuring the engine friction in a specific
way.

#### Stalls too easy

Disabling **Can Stall** ensures the engine never stalls.

Assuming Can Stall is enabled, you can:

- Reduce the **Stall Bias** value. Check out the calculated stall rpm value at the inspector, below
the engine graph.
- Set **Idle Control** to **Active**.
- Change the **Clutch Type** to **Torque Converter** in Clutch Settings. This clutch type doesn't
require to be actively engaged. Vehicle may stall at rare situations, like going backwards after
spin.

!!! Info "&fa-thumbs-o-up; Pro-Tip"
	Having a Torque Converter with a manual transmission is not a common setup but can greatly
	improve the gameplay. The user doesn't have to worry about clutch and still can apply throttle
	gently for maneuvering at low speeds.

#### How to ignite the engine at the beginning of the scene?

Two ways:

- Disable **Can Stall**. Engine will be always on, without possibility of stall.

- Keep sending the value "1" to `StdInput.Key` via the [Data Bus](databus-reference.md)
until the value `Vehicle.EngineStalled` returns "0". Then send a "0" to `StdInput.Key`.

	This procedure actually moves the ignition key to the ignite position. Once ignited (vehicle
	is no longer stalled) the key is moved back to normal operation.

#### How to configure the horsepower (HP)?

In the engine you can configure torque only, not HP. Torque is the actual value the engine transmits
to the components downstream. Engine power or HP is just the torque multiplied by the  angular
velocity. In the engine setup you specify directly:

- How much torque you want at the flywheel at two given rpm values, named _idle_ and _peak_.
- How the engine friction behaves with angular velocity.
- The rpm limit the engine is designed to operate at. The torque mechanically becomes zero at that
point.

The component calculates the engine values out of these settings. The torque curve (green curve in
the graph) is calculated by combining friction with the specified torque values and rpm limit.

The torques at the specified rpms can be extracted directly out of actual engine torque curves.
All you have to do is to adjust the friction curve until the actual torque curve and resulting HP
match the real data.

#### Too much engine friction makes the wheels rotate backwards

This might be either a correct behavior, or an indication that more integration steps/substeps are
required.

!!! Info "&fa-thumbs-o-up; Real driving tip"
	Releasing the throttle in a car driving forwards makes the engine friction to brake the vehicle.
	Too much engine friction can force the wheels actually spin at less speed than the road
	underneath, causing wheel lock. In racing cars this effect is typically observed when shifting
	down, and is compensated using the [heel-toe technique](http://www.drivingfast.net/car-control/heel-toe-shifting.htm).

A single wheel rotating backwards due to huge engine friction is likely caused by the
**differential**. This is a correct behavior. It's most likely to happen in open or
low-preloaded differentials, and specially when approaching a corner. The outer wheel getting
most of the weight combined with the huge engine brake could make the inner wheel to spin
backwards. Can be easily seen with a LEGO differential ;)

You can verify whether the behavior is correct or not by setting the differential type to
**Locked**. If now the wheels spin forwards when engine friction is applied, then the behavior
with the other differential type is correct.

If the effect can be observed even with a locked differential then try increasing the integration
substeps (up to 20, assuming a default Unity physics timestep of 0.02). Wheels should rotate
forwards but at less speed than the road when releasing the throttle. As extreme fix, you could
try setting the Unity physics time step to 0.01 and 10 integration steps in the Vehicle
Physics solver. Note that this will increase the overall impact of physics in the CPU usage.

---

## Tire friction

#### Slip ratio and slip angle?

**The fact** is that slip ratio and slip angle concepts are *not* required at all for simulating
tires and vehicles. Vehicle Physics Pro uses a tire simulation model that doesn't use these concepts
at all, yet providing fully accurate and realistic results. I've conceived a _slightly simplified_
model of a tire where all frictions, forces and torques can be calculated in a much easier and
coherent way.

There are lots of papers, docs, etc. about tires and slip ratio / slip angle. _"Don't reinvent the
wheel"_ is a good lesson, but this time I strongly believe that the wheel needed to be reinvented
for most racing games and vehicle simulations.

Here's a quote from my article [Facts and Myths of the Pacejka curves](http://www.edy.es/dev/2011/12/facts-and-myths-on-the-pacejka-curves/):

> The lateral force is based on the slip angle. However the slip angle does not account for any
> variation with the speed. Intuitively, the forces generated at high speed must be greater than
> the forces at low speed with the same slip angles. But V = (0.2, 0.1) results in the same slip
> angle (and thus the same force) as V = (20, 10).

This quote is based on Brian Beckman's thoughts about the Magic Formula not accounting for
any variation of the force with speed (reference: [The Physics Of Racing, chapter 22](http://phors.locost7.info/phors22.htm)).

In my opinion the slip ratio and slip angle concepts belong to the automotive and tire industries.
They are used to make studies, write standards, specifications, etc. but they hare hard to fit into
games and vehicle simulations in a coherent way. Slip ratio, slip angle and all their related
stuff are required if you need your simulation to match exactly the real specifications and product
details given by tire and automotive manufacturers, as they use these standards for exchanging
information.

For instance, if you want your simulation to reproduce exactly the tire model XXX from manufacturer
YYY then you need to use the slip ratio and slip angle concepts because the manufacturer will
provide the specifications and measures on that tire using these concepts and standards.

Even so, these standards are not applicable to many common situations such as low speeds and steady
states. These situations must still be "faked" and threated separately in the simulation. Racing
games are mostly in the "high speed" mode, so they don't care. But many common non-racing situations
produce incoherent results using the slip ratio and slip angle concepts. The tire model in Vehicle
Physics Pro provides perfectly coherent behaviors at any speed, with perfectly defined continuous
transitions between any state.

---

## Vehicle behavior

#### Too much understeer!

That will probably be the correct behavior for realistic settings. Most vehicles tend to understeer
as natural behavior.

I'd like to quote [this forum post from Stefano Cassillo](http://www.gamedev.net/topic/631886-car-physics-sharing-work-ideas-formulas-and-car-parameters/page-2#entry4986662),
developer of the [Assetto Corsa](http://www.assettocorsa.net) simulator, regarding lateral friction:

> Tire lateral grip
>
> It depends a lot.. **a typical road tire will give you 1.0 - 1.1 G..** with load sensitivity and all you should get close to 1.0G on a skidpad. A normal road car with 1.0G lateral acc is considered VERY good (ie. the BMW Mini scores a tad less than 1.0G and is considered a very good handling car).
>
> **Casual players usually find this VERY low..** this has lots of possible explanations:
>
> 1. No real experience with real cars.. and I mean track experience in a competitive way, not the kind of experience you get being an idiot on public roads.
>
> 2. The result of 1 is that no driving skills are developed... typical sign, Joe-I-Go-Fast goes too fast into a corner, the car will understeer and he turns more into the corner, making understeer worse and going even more wide into the wall.
>
> 3. They are used to other games either arcade a-la PGR or F1 games where cars have huge amount of grip.

Vehicle Physics Pro will implement these solutions:

Automatic steering angle limit
:	It will limit the maximum steering angle to the value that provides most lateral grip at the
	actual vehicle speed.

	In my opinion most of the understeer problem is a perception issue. Without steering wheel +
	force feedback there's no clear idea on when the tires are performing their maximum grip. By
	limiting the angle with speed you give the driver a clear perception on how much the vehicle
	can steer at the actual speed (with the actual tires). Thus, they learn to brake on sharp
	turns in order to allow more steering angle. Choosing the correct balance among speed & steering
	angle on each turn gives the vehicle the best racing line.

	This solution had been implemented in the former Edy's Vehicle Physics package under the driver
	aid "ESP".

Force feedback
:	Allowing the use of steering wheel controllers for feeling the actual grip of the tire while
	cornering.

Arcade mode
:	A component that will enforce the vehicle to steer no matter the actual tire settings. Should
	be used when this kind of gameplay is required by the project.

#### Drift settings

Use the same settings as real drift cars. This is quoted from [WhateverMan at gamedev.net](http://www.gamedev.net/topic/664785-car-physics-turbochargers-and-friends/#entry5205193):

> ALSO about drift cars, what I said earlier, 52/48 was 52% on the front and 48% on the back.
> Drift cars are Front Rear Layout cars. You need more weight on the front because you control
> with your wheels and lighter rear to get more wheel spin. Later you use rear wings aerodynamics
> to balance the angles with speed requirement.

---

## Custom components and vehicles

#### How to control the vehicle my way? (mobile controller, AI, ...)

Create your own input component, i.e. VPCustomInput, for sending your input values (throttle, brakes
steering...) to the vehicle via [Data Bus](databus-reference.md). Use the included standard input
component `VPStandardInput.cs` as example on how to send the values. Add your custom input component
to the vehicle GameObject instead of [VPStandardInput](../components/vehicle-input.md).

Do not modify the original scripts, as future updates may override your changes. The package is
designed so any functionality could be added via custom scripts.

---

