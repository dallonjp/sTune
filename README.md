# sTune    [![arduino-library-badge](https://www.ardu-badge.com/badge/sTune.svg?)](https://www.ardu-badge.com/sTune) [![PlatformIO Registry](https://badges.registry.platformio.org/packages/dlloydev/library/sTune.svg)](https://registry.platformio.org/packages/libraries/dlloydev/sTune)

This is an open loop PID autotuner using a novel s-curve inflection point test method. Tuning parameters are typically determined in about ½Tau on a first-order system with time delay. Full 5Tau testing and multiple serial output options are provided.

#### Reaction Curve Inflection Point Tuning Method

This tuning method determines the process gain, dead time, time constant and more by doing a shortened step test that ends just after the [inflection point](http://en.wikipedia.org/wiki/Inflection_point) has been reached. From here, the apparent maximum PV (input) is mathematically determined and the  controller's tuning parameters are calculated. Test duration is typically only ½Tau.

- See [**WiKi**](https://github.com/Dlloydev/sTune/wiki) for test results and more.

#### Inflection Point Discovery

Accurate determination of the inflection point was given high priority for this test method. To accomplish this, a circular buffer for the input readings is created that's sized to 10% of samples. The buffer is used as a moving tangent line where the "head" of the tangent is based on the average of all readings in the buffer and the "tail" is based on the oldest instantaneous value in the buffer. The tangent line slides along the reaction curve where where the head (leading point) moves smoothly but with moderate response. The tail (trailing point) of the tangent line is represented by instantaneous input readings that have occurred in the past (oldest buffer data point).

The slope of the tangent line is checked at every sample. When the sign of the change in slope changes (i.e. slope goes from increasing to decreasing or from decreasing to increasing), this is the point of inflection ([where the tangent turns red here](https://en.wikipedia.org/wiki/Inflection_point#/media/File:Animated_illustration_of_inflection_point.gif)). After 1⁄16 samples has occurred with the new slope direction, then it's known that the point of inflection has been reached. Final calculations are made and the test ends.

#### Inflection Point Tuning Method

![Reaction Curve](https://user-images.githubusercontent.com/63488701/147842980-01d12e68-ed80-486c-823e-fcfaa6b89b41.png)

#### Configuration

- First, the PID controller is placed in `manual` mode.

- The tuner action is set to `directIP` or `reverseIP`, then configure sTune:

- ```c++
  tuner.Configure(inputSpan, outputSpan, outputStart, outputStep, testTimeSec, settleTimeSec, samples);
  ```

- Its expected that the user is already familiar with the controller and can make a rough estimation of what the system's time constant would be. Use this estimate for the `testTimeSec` constant.

- The `samples` constant is used to define the maximum number of samples used to perform the test. To get an accurate representation of the curve, the suggested range is 200-500. 

- `settleTimeSec` is used to provide additional settling time prior to starting the test. 

- `inputSpan` and `outputSpan` represent the maximum operating range of input and output. Examples:

  - If your input works with temp readings of 20C min and 150C max, then `inputSpan = 130;`
     If the output uses 8-bit PWM and your using its full range, then `outputSpan = 255;`
     If the output is digital relay controlled by millis() with window period of 2000ms, then `outputSpan = 2000;`

- `outputStart` is the initial control output value which is used for the first 12 samples.

- `outputStep` is the stepped output value used for sample 13 to test completion.

- after test completion, the setup can be updated for the next test and `tuner.Configure()` can be called again.

#### Inflection Point Test

- The `outputStep` value is applied after 12 samples and inflection point discovery begins.

- Dead time is determined when the input has increased (or decreased) for 4 samples.
- When the point of inflection is reached, the test ends. The apparent `pvMax` is calculated using:

- ```c++
  pvMax = pvIp + slopeIp * kexp;  // where kexp = 4.3004 = (1 / exp(-1)) / (1 - exp(-1))
  ```

- The process gain `Ku` and time constant `Tu` are determined and the selected tuning rule's constants are used to determine `Kp, Ki and Kd`. Also, `controllability` and other details are provided (see comments in `sTune.cpp`).
- In the user's sketch, the PID controller is set to automatic, the tuning parameters are applied the PID controller is run.

### Functions

#### sTune Constructor

```c++
sTune(float *input, float *output, TuningRule tuningRule, Action action, SerialMode serialMode);
```

- `input` and `output` are pointers to the variables holding these values.
- `tuningRule` provides selection of 10 various tuning rules as described in the table below.
- `action` provides choices for controller action (direct or reverse) and whether to perform a fast inflection point test (IP) or a full 5 time constant test (5T). Choices are `directIP`, `direct5T`, `reverseIP` and `reverse5T`.
- `serialMode` provides 6 choices for serial output as described in the table below.

| Open Loop Tuning Methods | Description                                                  |
| ------------------------ | ------------------------------------------------------------ |
| `ZN_PID`                 | Open loop Ziegler-Nichols method with ¼ decay ratio          |
| `ZN_Half_PID`            | Same as `ZN_PID` but with all constants cut in half          |
| `Damped_PID`             | Can solve marginal stability issues                          |
| `NoOvershoot_PID`        | Uses C-H-R method (set point tracking) with 0% overshoot     |
| `CohenCoon_PID`          | Open loop method approximates closed loop response with a ¼ decay ratio |
| `ZN_PI`                  | Open loop Ziegler-Nichols method with ¼ decay ratio          |
| `ZN_Half_PI`             | Same as `ZN_PID` but with all constants cut in half          |
| `Damped_PI`              | Can solve marginal stability issues                          |
| `NoOvershoot_PI`         | Uses C-H-R method (set point tracking) with 0% overshoot     |
| `CohenCoon_PI`           | Open loop method approximates closed loop response with a ¼ decay ratio |

| Serial Mode     | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| `serialOFF`     | No serial output will occur.                                 |
| `printALL`      | Prints test data while settling and during the test run. A summary of results is printed when testing completes. |
| `printSUMMARY`  | A summary of results is printed when testing completes.      |
| `printDEBUG`    | Same as `printALL`but includes printing diagnostic data during test run. |
| `printPIDTUNER` | ➩  Prints test data in csv format compatible with [pidtuner.com](https://pidtuner.com/#/). <br />➩  Requires the controller `action` being set to `direct5T` or `reverse5T`<br />➩  Just copy the serial printer data and import (paste) into PID Tuner for further<br />      analysis, model identification, fine PID tuning and experimentation. <br />➩  Note that `Kp`, `Ti` and `Td` is also provided for PID Tuner. |
| `printPLOTTER`  | Plots `pvAvg` data for use with Serial Plotter.              |

#### Instantiate sTune

```c++
sTune tuner = sTune(&Input, &Output, tuner.ZN_PID, tuner.directIP, tuner.printALL);
/*                                         ZN_PID        directIP        serialOFF
                                           ZN_Half_PID   direct5T        printALL
                                           Damped_PID    reverseIP       printSUMMARY
                                           NoOvershoot_PID reverse5T     printDEBUG
                                           CohenCoon_PID                 printPIDTUNER
                                           ZN_PI                         serialPLOTTER
                                           ZN_Half_PI
                                           Damped_PI
                                           NoOvershoot_PI
                                           CohenCoon_PI
*/
```

#### Configure

```c++
void Configure(const float inputSpan, const float outputSpan, float outputStart, float outputStep,
uint32_t testTimeSec, uint32_t settleTimeSec, const uint16_t samples);
```

This function applies the sTune test settings.

#### Set Functions

```c++
void SetControllerAction(Action Action);
void SetSerialMode(SerialMode SerialMode);
void SetTuningMethod(TuningMethod TuningMethod);
```

#### Query Functions

```c++
float GetKp();                 // proportional gain
float GetKi();                 // integral gain
float GetKd();                 // derivative gain
float GetProcessGain();        // process gain
float GetDeadTime();           // process dead time (seconds)
float GetTau();                // process time constant (seconds)
float GetTimeIntegral();       // process time integral (seconds)
float GetTimeDerivative();     // process time derivative (seconds)
uint8_t GetControllerAction();
uint8_t GetSerialMode();
uint8_t GetTuningRule();
void GetAutoTunings(float * kp, float * ki, float * kd);
```

#### Controllability of the process

```c++
float controllability = _Tu / _td + epsilon;
```

When the test ends, sTune determines [how difficult](https://blog.opticontrols.com/wp-content/uploads/2011/06/td-versus-tau.png) the process is to control.

| Controllability | Comment                 |
| --------------- | ----------------------- |
| Above 0.75      | Easy to control         |
| 0.25 to 0.74    | Average controllability |
| 0.00 to 0.24    | Difficult to control    |

#### References

- [Comparison of PID Controller Tuning Methods](http://www.ie.tec.ac.cr/einteriano/analisis/clase/13.1.Zomorrodi_Shahrokhi_PID_Tunning_Comparison.pdf)
- [Ziegler-Nichols Open-Loop Tuning Rules](https://blog.opticontrols.com/archives/477)
- [Inflection point](https://en.wikipedia.org/wiki/Inflection_point)
- [Time Constant (Re: Step response with arbitrary initial conditions)](https://en.wikipedia.org/wiki/Time_constant)
- [Sample Time is a Fundamental Design and Tuning Specification](https://controlguru.com/sample-time-is-a-fundamental-design-and-tuning-specification/)

