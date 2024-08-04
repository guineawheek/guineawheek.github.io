# Event loops in robotics programming

very rambly and very WIP document

## Imperative programming

So a lot of people when they get introduced to programming, they understand it as a series of instructions that a computer follows. Print "Hello World". For each number from 1 to 100, print "Fizz" if it's divisible by 3 and "Buzz" if it's divisible by 5 and "FizzBuzz" if it's divisible by 15. The computer follows the series of instructions linearly from start to finish. 

This works in your introductory programming classes, and it even works in FIRST Lego League. Your FLL robot might have programs that at a high level are basically:

```
start
go forward 5 turns
turn in-place until the gyro reads 35 degrees
go forward 2 turns
deposit the payload
turn in-place another 40 degrees
go forward 10 turns
end
```

And these programs are pretty easy to understand, even if they occasionally have control flow like `if` statements or `for` loops. It's all well and good.

At least until you end up in FTC or FRC and you're slapped in the face with a brand new concept:

## The event loop

In FLL, you don't directly control the robot when it moves on the field. It generally follows a preprogrammed laundry list of instructions until it finishes or you pick it up off the field and get penalty points.

But in the big boy programs suddenly you're faced with teleop mode and now your robot has to do what any self-respecting robot is able to do: react to its environment dynamically. More specifically, you're typically given some setup like this:

```java

/**
 * Method called by the robot every 20 milliseconds while the robot is enabled
 */
public void loop() {
    // YOUR CODE HERE
}
```

where a `loop()` function is called repeatedly to update what the robot is up to. Now you have to deal with your code being called repeatedly all the time. At least unless you want to copypaste your robot's code millions of times and hope you don't "run out of code". 

The official examples on how to get your robot moving are helpful and walk you through how to make your robot drive. Eventually you end up with code that looks something like this for your differential (tank) drive:

```java
public void loop() {
    leftDriveMotor.setPower(gamepad1.getLeftStickY());
    rightDriveMotor.setPower(gamepad1.getRightStickY());
}
```

Now when you press forward on the left or right sticks the left or right side of the drivetrain moves forward.
Intuitively this makes sense; the robot's drivetrain motor powers are being continuously updated with what you press on the gamepad. 

You're able to now drive your robot around. You even take this concept of reacting to the gamepad's state and extend it a bit further so your robot's arm and end claw can now work:


```java
public void loop() {
    // Update the drivetrain
    leftDriveMotor.setPower(gamepad1.getLeftStickY());
    rightDriveMotor.setPower(gamepad1.getRightStickY());

    // Update the arm
    armMotor.setPower(gamepad2.getLeftStickY());
    // Update the claw
    if (gamepad2.a) {
        // Close the claw
        clawServo.setPositionDegrees(30);
    } else {
        // Open the claw
        clawServo.setPositionDegrees(180);
    }
}
```

This is great and all but you notice that holding the arm at a specific angle requires some precision on Driver 2's gamepad. You don't really worry about this with the claw servo because that's a hobby servo that you can directly command to a position and the servo will figure it out.

It's okay though, you set up PID on the motor controller so now you can make the arm angle scale with the position of the left stick with
```java
armMotor.setPosition(gamepad2.getLeftStickY() * SCALING_FACTOR);
```

But then the drivers ask you to perform a more complicated task: they want it such that when a game element is detected in the claw, that the claw automatically closes and the arm lifts up to the deposit position, and when you press the left bumper, the claw opens again and the arm goes back to the intake state

This seems easy conceptually, and you already did something like this in FLL. Coceptually the ocode is simple, right?

```java
public void loop() {
    // Update the drivetrain
    leftDriveMotor.setPower(gamepad1.getLeftStickY());
    rightDriveMotor.setPower(gamepad1.getRightStickY());

    if (clawSensor.detectsElement()) {
        clawServo.setPositionDegrees(30);
        // wait for the claw to close securely
        Thread.sleep(500);
        // set the arm motor to deposit
        armMotor.setPosition(DEPOSIT_POSITION);
    }

    if (gamepad2.leftTriggerPressed()) {
        // open claw
        clawServo.setPositionDegrees(135);
        // wait for the element to fall out
        Thread.sleep(500);
        // Update the arm
        armMotor.setPosition(INTAKE_POSITION);
    }
}
```

And this code works. But you notice that you can't control the robot's drivetrain while the claw opens and closes. 
But you make do, because it's only half a second, who cares? 

You tell the drivers that's just how the code works, but thing is, **you still care just a little bit.**

## The concurrency problem

### The arm extension

The first competition goes okay. Your robot works surprisingly well for an inexperienced team. It's great, you even make it to elims, at least until your robot tips over in semifinals.

Following the #tipgate scandal, build team decides they are going to make the arm a telescoping extension so that the arm sticks out less when rotating and moving around. This makes the moment of inertia lower and thus the overall robot is less tippy. You revise your code to adjust for this new development:

```java
public void loop() {
    // Update the drivetrain
    leftDriveMotor.setPower(gamepad1.getLeftStickY());
    rightDriveMotor.setPower(gamepad1.getRightStickY());

    if (clawSensor.detectsElement()) {
        clawServo.setPositionDegrees(30);
        // wait for the claw to close securely
        Thread.sleep(500);

        // Retract the arm extension
        armExtensionMotor.setPosition(RETRACT_POSITION);

        // Wait for the arm extension to retract fully
        while (armExtensionMotor.atPosition(RETRACT_POSITION)) { Thread.yield(); }

        // set the arm motor to deposit
        armPivotMotor.setPosition(DEPOSIT_POSITION);
    }

    if (gamepad2.leftTriggerPressed()) {
        // Extend the arm extension
        armExtensionMotor.setPosition(EXTEND_POSITION);

        // Wait for the arm extension to extend fully
        while (armExtensionMotor.atPosition(EXTEND_POSITION)) { Thread.yield(); }

        // open claw
        clawServo.setPositionDegrees(135);
        // wait for the element to fall out
        Thread.sleep(500);

        // Retract the arm extension
        armExtensionMotor.setPosition(RETRACT_POSITION);

        // Wait for the arm extension to retract fully
        while (armExtensionMotor.atPosition(RETRACT_POSITION)) { Thread.yield(); }

        // Update the arm
        armMotor.setPosition(INTAKE_POSITION);
    }

    // Allow the robot to intake
    if (gamepad2.rightTriggerPressed()) {
        // Pivot the arm
        armPivotMotor.setPosition(INTAKE_POSITION);

        // Extend the arm extension
        armExtensionMotor.setPosition(EXTEND_POSITION);
    }
}
```

This still works, but the added delay of extending and retracting the arm is really starting to get on the drivers' nerves. They can't control the drivetrain while the arm is moving and it's increasing cycle times. The robot controller keeps throwing errors that loop times are getting overrun. This just isn't working. 

### Imperative just doesn't work here

This is the approximate point that a lot of junior robotics programmers struggle with -- you just can't directly translate imperative code to this event loop structure. Instead we have to re-think how we reason about robot code. Instead of thinking of our `loop()` function as just how we write code that gets executed in order, we have to think of it as where we define how the robot should react to changes in its environment -- be it from sensors or inputs from the drivers.

The same concept exists in video game programming -- a player character entity has to update its state every frame based on things like its health, interactions with the game environment, and player inputs. We're dealing with an update loop. But what does the loop update? It updates the robot or player's **state.** So when we write code in an update loop we have to think about how we represent and change that state.

Fortunately, there's a very well known paradigm to do _exactly_ this. 

## State machines

State machines are a common paradigm for representing things that need an update loop. In a state machine we define two broad categories of things: 
* the states our Thing can be in
* under what conditions our Thing can transition from one state to another.

The current state we're in might define what power we set our motors or how else we choose to change our environment. 
It may also just define what next actions and events our Thing is currently able to react to.

We can draw the relationships between our states and the transition conditions using something called a "state-transition diagram". 

### Example: automatic sliding door

Let's say we're tasked with building an automatic sliding door for local buildings. Let's think about what states a door can be in.
The door obviously must be able to be open and closed, but since the door doesn't instantaneously open or close, we can consider "opening" and "closing" to both be states too. So our states can be described as:
* fully closed
* fully open
* opening
* closing

We can also define how each state can transition into each other:
* When the door detects a person and is fully closed, the door starts opening.
* When the door is opening but sensors detect it's opened all the way, the door is fully opened.
* If the door is fully opened but there's inactivity after a period of time, the door will start to close.
* When the door is closing and sensors detect it's closed all the way, the door is fully closed.
 * If a person is detected while the door is closing, start opening the door instead.

We can draw up a state transition diagram that looks something like this:

```
[ fully closed ] -- person detected --+-> [ opening ]
     ^                               /        |
  door shut          person  _______/     door open
  detected   ______/ detected             detected
     |      /                                 v
[ closing ]  <---- inactivity  -----   [ fully open ]

```

Language features such as `switch` statements and enums usually map pretty well to making state machines.
We can switch on a variable that stores the current door's state and for each state perform the relevant action
and check for state transition conditions.

If we were writing the code to operate a door it might look something like:

```java

public enum DoorState {
    FULLY_CLOSED,
    FULLY_OPEN,
    OPENING,
    CLOSING,
}
DoorState state;
long closeTimeout;

// Assume this is called very frequently
public void loop() {
    switch (state) {
        case FULLY_CLOSED:
            if (personDetected()) {
                // We need to open the door, so we transition to the opening state.
                state = DoorState.OPENING;
            } else {
                // No state transition.
                // Stop/keep the door from moving
                doorMotor.stop();
            }
            break;
        case FULLY_OPEN:
            if (personDetected()) {
                // Reset the door close timer
                closeTimeout = System.currentTimeMillis();
            } else if ((System.currentTimeMillis() - closeTimeout) > 10_000) {
                // 10 seconds have passed without seeing someone,
                // transition state to closing.
                state = DoorState.CLOSING;
            } else {
                // Stop/keep the door from moving
                doorMotor.stop();
            }
            break;
        case OPENING: 
            if (doorOpenLimitSwitch.isActive()) {
                // The door has fully opened, transition to fully open.
                state = DoorState.FULLY_OPEN;
                // Also reset the close timeout timer to keep the door open initially.
                closeTimeout = System.currentTimeMillis();
            } else {
                // open the door.
                doorMotor.setPower(OPENING_POWER);
            }

            break;
        case CLOSING:
            if (personDetected()) {
                // Start opening the door again if we see someone, by transitioning to the opening state.
                state = DoorState.OPENING;
            } else if (doorCloseLimitSwitch.isActive()) {
                // The door has closed all the way, transition to the fully closed state
                state = DoorState.FULLY_CLOSED;
            } else {
                // No state transition.
                // Energize the door motor to close.
                doorMotor.setPower(CLOSING_POWER);
            }
            break;
    }
}
```

For each possible state the door can be in, we first check if we should switch states based on current conditions, and if we are to stay in the current state, execute that state's actions. Of course, this can get messy quicky and devolve into a massive, hard-to-read switch statement, so this is a prime candidate for splitting code into smaller methods:


```java
public void transitionToFullyOpen() {
    // Update the variable that tracks what state we're in
    state = DoorState.FULLY_OPEN;
    // instantaneously stop the motor
    doorMotor.stop();
    // Also reset the close timeout timer to keep the door open initially.
    closeTimeout = System.currentTimeMillis();
}

public void transitionToFullyClosed() {
    // instantaneously stop the motor
    doorMotor.stop();
    state = DoorState.FULLY_CLOSED;
}

public void transitionToOpening() {
    // instantaneously start the motor
    doorMotor.setPower(OPENING_POWER);
    state = DoorState.OPENING;
}

public void transitionToClosing() {
    // instantaneously start the motor
    doorMotor.setPower(CLOSING_POWER);
    state = DoorState.OPENING;
}

public void handleFullyClosed() {
    if (personDetected()) {
        // We need to open the door, so we transition to the opening state.
        transitionToOpening();
    } else {
        // No state transition.
        // Stop/keep the door from moving
        doorMotor.stop();
    }
}

public void handleFullyOpen() {
    if (personDetected()) {
        // Reset the door close timer
        closeTimeout = System.currentTimeMillis();
    } else if ((System.currentTimeMillis() - closeTimeout) > 10_000) {
        // 10 seconds have passed without seeing someone,
        // transition state to closing.
        transitionToClosing();
    } else {
        // Stop/keep the door from moving
        doorMotor.stop();
    }
}

public void handleOpening() {
    if (doorOpenLimitSwitch.isActive()) {
        transitionToFullyOpen();
    } else {
        // open the door.
        doorMotor.setPower(OPENING_POWER);
    }
}

public void handleClosing() {
    if (personDetected()) {
        // Start opening the door again if we see someone, by transitioning to the opening state.
        transitionToOpening();
    } else if (doorCloseLimitSwitch.isActive()) {
        // The door has closed all the way, transition to the fully closed state
        transitionToFullyClosed();
    } else {
        // No state transition.
        // Energize the door motor to close.
        doorMotor.setPower(CLOSING_POWER);
    }
    break;
}

// the actual loop can be pretty short.
public void loop() {
    switch (state) {
        case FULLY_CLOSED:
            handleFullyClosed();
            break;
        case FULLY_OPEN:
            handleFullyOpen();
            break;
        case OPENING: 
            handleOpening();
            break;
        case CLOSING:
            handleClosing();
            break;
    }
}
```

Now you might ask, what are the benefits of doing things like this? Well, it means that when we detect someone trying to walk in while the door is closing we can react to it immidiately in a way that's harder for an imperative paradigm to do without some ugly hacks. It also means we can interleave different things going on at once; if our door controller is suddenly now responsible for an LED display, it can update the LED display's state machine in `loop()` without either interfereing with each other because they don't wait on each other. 

## Applying this back to the armbot example
todo

## Conclusion
wpilib's command based framework models this state-machine framework, it just doesn't really say so.

### Alternate approaches: async/await

State-machine code can still end up fairly cumbersome -- you may forget to fill out all parts of a switch case somewhere or forget a state diagram somewhere.

It also makes reasoning about linear execution harder because now you have to think about how execution flows through your state machine. A lot of modern concurrency frameworks prefer a "green thread" or "async/await" approach where normally blocking operations are turned into futures that you can "await" on and normally blocking code (e.g. waiting for a timeout or some sensor condition to hold) can have its execution suspended while other tasks run simultaneously. 

One FRC team uses [Lua coroutines](https://bvisness.me/coroutines/) to re-linearize their code.

The programmer of an FTC team wrote this blog post vehemently arguing in favor of ["structured concurrency"](https://max.xz.ax/blog/structured-concurrency-robot-control/) instead of dealing with state machines. 

I don't know how this can be neatly folded back into the Java monoculture of FIRST while also being accessible to students (and I personally believe that robot programming languages should be strongly and statically typed), but I do think there's something to be said for the idea, because being able to write something like

```java
if (gamepad2.leftTriggerPressed()) {
    // open claw
    clawServo.setPositionDegrees(135);
    // wait for the element to fall out, asynchronously
    asyncSleep(500);
    // Update the arm
    armMotor.setPosition(INTAKE_POSITION);
}
```
and not freeze up your drivetrain could be a lot more intuitive for newer programmers while teaching a lot about how concurrency works in a lot of modern network/higher level development.

I think a big problem is that people going into write async programming frameworks for Java already have a good handle on how programming works so they lean in quite a bit into lambdas/closures and other advanced language features when many students don't even fully understand how OOP works, so there's definitely some room to explore here. 

Chances are any sort of "async" programming in Java would likely end up being backed by threads and thread synch primitives which have their own issues (mostly OS context switching overhead) but like, if you seriously care about that you're probably not doing this anyway.

Maybe I should take a crack at a java async framework...