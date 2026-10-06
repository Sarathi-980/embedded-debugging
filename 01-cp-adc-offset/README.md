# Case 01: ADC Reads Low on an EV charger's Control Pilot

An STM32 ADC reading came out lower than the circuit calculations predicted. This is how i traced it from the firmware back to hardware, and what I learned along the way.

## The Problem

On an EV charger board, the STM32F103 measures the Control Pilot (CP) signal through a resistor divider into its ADC. The firmware I inherited
converted the ADC counts back to CP voltage, but the results didn't match the real signal, which I confirmed with a multimeter.

## Background: The Control Pilot Signal

- The Control Pilot signal is an EV charger and a car used to communicate (IEC 61851-1 / SAE J1772):

- The charger(EVSE) drives a **+12V, 1 kHz square wave** on the CP line. 
- The **car changes the voltage level** with its own resistors, to signal its state: 

| CP voltage | State | Meaning |
|---|---|---|
| +12 V | A | No car connected |
| +9 V | B | Car connected, but not ready |
| +6 V | C | Car ready, charging |
| +3 V | D | Charging, ventilation required |
| 0 V | E | Short circuit or no power, Error |
| -12 V | F | Charger fault |

- The -12 V level has second role, it the **low half of every PWM cycle**. The car has a diode on the CP line, so it can only pull down the positive half. If the doesn't read -12 V, the diode is missing and the charger must not charge.
- The **charger set duty cycle (PWM width) to tell the car the maximum current it may draw from charger.

The MCU ADC only accepts 0-3.3V, so a resistor dividor must maps the full **the -12 V to +12 V signal** range into the window. That divider is where this story happens.

## Understanding the Circuit

- Lets take a example circuit for CP ADC conversion.
- The divider has three resistors meeting at one node, which connects the ADC pin:
  CP (±12 V)-----[R1 300k]------[R3 82k]------3.3 V (Pull up)
  |                         |
  |-----------------------> ADC pin (Vn) 
  |
  [R2 100k]
  |
  GND

### Step 1: KCL at the node n

- Krichoff's current law says all the current flowing into a node must sum to zero. we assume every current flow into node, So no need to guess directions. No problem on this assumption because current flows out of the node, will comes out negative.

    (Vcp - Vn) / R1 + (3.3 - Vn) / R3 + (0 - Vn) / R2 = 0

Solving for Vn:
    Vn = 0.1306 × Vcp + 1.576 V

From above equation we can see how Control Pilot Voltage (Vcp) gives node voltage Vn, Gain controls Vcp signal and then the 3.3 V source signal adds offset to it.
- **Gain (0.1306):** how much the ADC pin moves per 1 V change on CP signal.
- **Offset (1.576):** where CP = 0 V lands on the ADC pin.

### Step 2: The intuitive view (Superposition)

- The simpler method by looking source separately.

**Offset:** set CP to 0 V. Then R1 acts as another resistor to ground, so pull up resistor R3 forms a simple divider with R1 || R2. R1 parallel to R2 gives total resistance of 75K.
    Offset = 3.3 × 75 / (82 + 75) => 1.576 V

**Gain:** the 3.3 V rail never changes, so for *changes* on CP it behaves like ground. Now R3 is parallel with R2 (total resistance 45.05k):
    Gain = 45.05 / (300 + 45.05) = 0.1306

This showed me what actually resistor does. The pull up R3 creates the offset, and it keeps -12V from going below 0 V at the ADC pin. without it, the MCU would see the negative voltages (which is not good for MCU).

### Expected Readings

| CP voltage | ADC pin | ADC counts (12 bits => (ADC voltage / 3.3) * 4095 |
| --- | --- | --- |
| +12 V | 3.143 V | 3900 |
| +9 V | 2.751 V | 3414 | 
| +6 V | 2.360 V | 2928 |
| 0 V | 1.576 V | 1956 |
| -12 V | 0.009 V | 11 | 

## Fixing the Existing Conversion Formula

The firmware I inherited converted ADC counts to CP voltage using:
    Vcp = counts × 12 / 4095

when I tested it at known points, the readings didn't match:

| ADC counts | True CP voltage | The existing formula result | 
|---|---|---|
| 3900 | +12 V | 11.43 V |
| 1956 | 0 V | 5.73 V |
| 11 | -12 V | 0.03 V |

From circuit analysis, the issues are clear:

1. **12 is not the reference.** The ADC reference is 3.3 V. The formula based on 12 V only works if for plain 0 V to +12 V with no pull-up, where Vref ÷ gain is equals to 12 V.
2. **No offset term.** The pull-up lifts readings by the same offset. so 0 V on CP reads 1956 counts, not 0. The above formula didn't account for that. 
  
### The Correct Formula

Reversing the divider takes two steps:

    1. counts to pin voltage => Vn  = (counts / 4095) × 3.3
    2. pin -> CP voltage =>     Vcp = (Vn - 1.576) / 0.1306     (from previously derived formula)

Substitute the Vn in 2nd gives, Vcp = counts * 0.00617 - 12.06

The formula gives follwing results:

| ADC counts | True CP voltage | Correct formula result |
|---|---|---|
| 3900 | +12 V | 12.00 V |
| 1956 | 0 V | 0.00 V |
| 11 | -12 V | -12.00 V |

with the formula fixed, I expected the board to read +12 V. But it printed **11.47 V**. I added the raw ADC counts to the logs, which showed **3814 counts** instead of expected 3900. Since the formula was correct, the difference had to come from the hardware.

## Investigation : Measuring Stage by Stage

Instead of guessing I followed the signal from source to the ADC and compared each measurement with the calculated value.

| Step | What I measured  | Expected | Multimeter measurement | Conclusion |
|---|---|---|---|---|
| 1 | CP output | 12 V | 12 V | Op-amp stage is fine |
| 2 | Divider node (ADC pin) | 3.14 V | ~3.07 V | The node voltage is low |
| 3 | Pull-up rail | 3.3 V | ~3.15 V | The pull-up rail is low |
| 4 | MCU VDDA pin | 3.3 V | 3.3 V | The ADC reference is correct |

From step 2, we can say the node voltage itself is low, so the ADC was **reading correctly**. But problem was in the hardware, not the firmware.

Scaling the offset to the real rail (3.15 V):

    Offset = 3.15 V × 75 / (82 + 75) => 1.505 V
    Vn = 0.1306 * 12 V + 1.505 = 3.072 V
    Counts = (3.072 / 3.3) * 4095 = 3812

The calculated 3812 matches with the measured 3814 within couple of counts. 
A pull-up rail is just 0.15 V low is explain the error.


## Root Cause

1. The formula didn't account the offset based calculation for CP voltage.

2. The design **(schematic diagram) assumed the pull-up rail and the ADC reference (VDDA) were both exactly 3.3V**. but on the real board:
    - **VDDA is 3.3 V** so the ADC was reading correctly
    - **Pull-up rail is 3.15 V**, so the divider offset is lower than designed.

The pull-up sets the offset, so a lower rail shifted CP voltage reading down by 0.5 V.

**The firmware wasn't wrong and ADC in chip also wasn't wrong. The assumption that two rails equals were wrong.**

## Adapting (Calibrating) the Formula for the Board

Since this is testing board, until next hardware, I use the calibrated formula for this board.

The correct formula earlier assumes pull-up rail is 3.3 V. For this board, the offset is 1.505 (we previously calculated).
    
    Vcp = (Vn - 1.505) / 0.1306

Combined with counts -> pin voltage (substitute the Vn gives)

    Vcp = counts * 0.00617 - 11.52

| ADC counts | True CP voltage | Board formula result |
|---|---|---|
| 3814 | +12 V | 12.01 V |
| 1867 | 0 V | 0.00 V |

### A mistake I made during debugging: 
    I used measured pull-up rail 3.2 V for the slope as well:
        
        Vcp = counts × 0.00598 - 11.52  ==>  read 11.29 V at 12 V

I used AI to help analyze where my formula went wrong, by giving the measurements I did in this board using multimeter. From that I learned VDDA is 3.3 V means the ADC reference is working correctly with 3.3 V.
 
- **Slope** comes from **VDDA**, the ADC reference: 3.3 / 4095 / 0.1306 = 0.00617
- **Offset** comes from **pull-up rail**: 1.505 / 0.1306 = 11.52

So both are independent.    

## Scaling Up: Two-Point calibration

For this test board, I calculated the formula manually from my measurements. That works for one board.
When multiple boards are built, I plan to use **two pointer calibration**, measure two known CP voltages on each board and let the firmware calculate its own line.

1. Short CP to ground 0 V and record the ADC counts.
2. Hold CP at +12 V and record the ADC counts.
3. Store both values and convert every reading with:
    Vcp = (counts - counts at 0 V) × 12 / (counts at 12 V - counts at 0 V)
        
A straight line is defined by two points here, 0 V and 12 V. On this board at 0 V the counts is 1867 and at 12 V the counts is 3814, 
    => Vcp = (counts - 1867) × (12 / 1947) => (counts - 1867) × 0.00616 V. here slope is same as formula. 
    from this we can get value of the voltage from counts using MCU calculation without using multimeter measurement.

## What I learned from this

**1. HAL hides registers, not the physics.** If firmware was giving wrong counts value than expected, then understanding the circuit is the only way to find out what is going on. 

**2. Verify the inherited code against known points.** The existing firmware fails at 0V and negative voltage, because they didn't account offset determined by hardware design.

**3. Measure stage by stage to track the issue.** one measurement at each part ruled out each part of the circuit.

**4. A voltage divider is a straight line with gain and offset.** with KCL, I could see what resistor does. The pull up resistor sets offset, which keeps -12 v and 0 V at the ADC pin. CP's share of the node sets the gain.

**5. We need to know what are dependent.**. I had measured both rails, but used the wrong one in the slope, but slope depends on VDDA.

**6. Theory and measurement should agree.**
The calculation predicted 3812 counts, and the board measured 3814. When they match, you know you understand the circuit, not just the symptom.
 
**7. Pick the fix that fits the situation.**
A manual formula is fine for one test board. If multiple boards are there, we should do two-point calibration here, which absorbs every board's errors automatically, but with clear documentation.
 
**9. Find the root cause instead of adding a correction factor.**
A fudge factor would have made this one board read 12 V and broken the next one. After I found the issue I reported the issue to the hardware team.


## Code

A generic example of implementation in code, it takes the raw 12 bit ADC counts and return the CP voltage in volts.

```c

/* Ideal board (pull up rail == VDDA == 3.3 V)
   Vcp = counts × (3.3 / 4095 / 0.1306) − (1.576 / 0.1306)
*/
float cp_volts_ideal(uint16_t counts) 
{
    return counts * 0.00617f - 12.06f;
}

/* Test board, calculated manually from measurements:
 *    VDDA = 3.3 V (slope), pull-up rail = 3.15 V (offset)
 */
float cp_volts_board(uint16_t counts)
{
    return counts * 0.00617f - 11.52f;
}

/* Two-point calibration: works on any board.
 *    counts_0v and counts_12v are measured on each board and stored.    
*/
typedef struct {
    uint16_t counts_0v;     // ADC counts for 0 V
    uint16_t counts_12v;    // ADC counts for +12 V 
} cp_calib_t;

float cp_volts_calibrated(uint16_t counts, const cp_calib_t *cal)
{
    return ((float)counts - cal->counts_0v) * 12.0f
           / (float)(cal->counts_12v - cal->counts_0v);
}
```

