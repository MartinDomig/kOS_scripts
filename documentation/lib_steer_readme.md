## lib\_steer (steering library)

### Description

This library contains code used to control and check kOS's cooked steering.

### Global variable reference

#### `STEER_ON`

A boolean that is used to indicate whether we are currently locking the steering to anything or not.

### Function reference

#### `isSteerOn()`

Returns the value of `STEER_ON`.

#### `steerOff()`

Disables the steering (unlocks the steering and sets `STEER_ON` to `FALSE`).

#### `steerTo(fore_function, top_function)`

If not specified, `fore_function` defaults to the craft's current fore vector: `{ RETURN FACING:FOREVECTOR. }` and `top_function` defaults to the craft's current top/up vector: `{ RETURN FACING:TOPVECTOR. }`

This calls: `LOCK STEERING TO LOOKDIRUP(fore_function(),top_function()).`

The function also sets `STEER_ON` to `TRUE` and stores the current time by calling: `setTime("STEER").`

Note - the steering uses functions as parameters because these will update each time they are called. If we passed in a plain vector, this would not update over time.

#### `steerSurf(prograde)`

Locks the steering to the surface prograde (or retrograde) vector.

If not specified, the default value for `prograde` is `TRUE`.

This calls `steerTo()`, passing in an anonymous function that returns the craft's surface prograde or surface retrograde vector, depending on the value of `prograde`.

#### `steerOrbit(prograde)`

Locks the steering to the orbital prograde (or retrograde) vector.

If not specified, the default value for `prograde` is `TRUE`.

This calls `steerTo()`, passing in an anonymous function that returns the craft's orbital prograde or orbital retrograde vector, depending on the value of `prograde`.

#### `steerNormal()`

Locks the steering to the orbital normal vector, with the "top" of the craft rotated to face the Sun.

This calls `steerTo()`, passing in an anonymous function that returns the craft's orbital normal vector, and one that returns the position of the Sun.

#### `steerSun()`

Locks the steering to face the craft at the Sun.

This calls `steerTo()`, passing in an anonymous function that returns the position of the Sun.

#### `steerAV()`

A craft will continually, slowly rotate as a result of motion around its parent body. This function calculates and returns the current 'expected' angular velocity (in radians per second) of the active vessel.

In an orbit, the prograde vector moves relative to the universal reference vector as you orbit a body (e.g. if you face prograde then engage time warp for half an orbit, you'll be pointing at the retrograde marker). By taking the predicted velocity now and in one second's time, we can estimate the current angular velocity by taking the angle between the two.

On the ground, the active vessel's facing will rotate as a result of the rotation of the planet. This will be constant, based on how long it takes the planet to rotate.

#### `steerOk(allowed_angle, angular_vel_precision, timeout_seconds)`

This function returns `TRUE` or `FALSE` depending on whether the steering is pointed in the right direction or not. It is typically called as part of a WAIT UNTIL: `WAIT UNTIL steerOk().`

The logic is as follows:

    IF diffTime("STEER") <= 0.1 { RETURN FALSE. }
We deliberately don't check the steering if we are within a tenth of a second of having locked steering to a new direction. This gives the steering manager time to enable itself.

    IF NOT STEERINGMANAGER:ENABLED { hudMsg("ERROR: Steering Manager not enabled!"). }
This is a problem. We should never call `steerOk()` unless we've locked the steering (a check on `STEER_ON` was removed to save a bit of space), in which case `STEERINGMANAGER:ENABLED` should always be `TRUE`. However, the steering manager can get confused in a few situations (repeated docking/undocking is the way I ran into this). In that situation, we have an error that a kOS script can't handle very well. Switching back to the space centre than back to the craft seemed to be the only solution that would clear it up; just rebooting didn't help. As such, we have a rare call to `hudMsg()` to print an error in the middle of the screen in the knowledge that the next step will cause the script to crash.

    IF VANG(STEERINGMANAGER:TARGET:VECTOR,FACING:FOREVECTOR) < aoa AND
       SHIP:ANGULARVEL:MAG * angular_vel_precision / 10 < MAX(steerAV(), 0.0005) {
      pOut("Steering aligned.").
      RETURN TRUE.
    }
This checks the angle between where we are facing and where we have asked the steering manager to face, we want this to be less than the `allowed_angle` parameter. Previous versions of the script used this to set off a timer, only returning `TRUE` once we had been facing the right direction for a few seconds. Instead of that, we now check the magnitude of the craft's angular velocity. If this drops below our required minimum, we return `TRUE`.

    IF diffTime("STEER") > timeout_seconds {
      pOut("Steering alignment timed out.").
      RETURN TRUE.
    }
If we have not returned a value yet, we check the time since steering was enabled. If this is greater than the `timeout_seconds` parameter, we give up and return `TRUE`. It's possible we are pointed in the right direction, but the steering is wobbling too much for the angular velocity to drop below the threshold. I have noticed this with small probes coupled with a powerful reaction wheel.

Otherwise, we return `FALSE`. The expectation is that we will keep calling this function each tick until the return value is `TRUE`.

If not provided, the default values are:
* `allowed_angle` : `1`
* `angular_vel_precision` : `4`
* `timeout_seconds` : `60`

#### `dampSteering()`

This function is used to dampen out any current oscillations or rotation, without pointing in any particular direction. It is commonly called after the steering has been pointed at a manoeuvre node. Simply unlocking the steering would not be very effective as the craft would maintain any rotation that it currently has.

This takes a copy of the craft's current facing vector, locks the steering to it and waits until `steerOk()` returns `TRUE`.


Geoff Banks / ElWanderer
