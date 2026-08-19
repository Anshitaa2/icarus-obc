# Icarus OBC Debugging Documentation

## 1. Overview
My understanding of the problem is that it is a fake flight software program. The computer repeats steps like reading sensors
and updating batteries over and over again- and each repetition is called a tick- which happens 2250 times (15 orbits x
150 ticks per orbit) in this particular program. With each tick, the scheduler runs the tasks, updates physical components 
and then logs it. 
## 2. Theory/Code learned
AppState contains all of the spacecraft's current information, so we use the keyword state to access information
from there.
Telemetry is the process by which the OBC sends information back to the ground. A telemetry frame is a C struct containing
one snapshot of sensor information. These frames are stored in a queue.
## 3. Process
I started by looking at the simulation output because I didn't understand the codebase. When i noticed values like SAFE 
changing and THERM+STALE becoming 1, I traced those variables back to the code to find where they were being changed. I used
AI to targetedly search the relevant functions.
Initially, I had some issues with understanding the output, so I inspected the scheduler, thermal sensor, ADC and actuator
code seperately. I also repeatedly ran the simulation to check whether the unexpected behavior still occurred. 
For some issues, I used AI to understand the code and identify possible fixes, following which I implemented the changes
and ran and reran the code. My understanding of each change is as follows:
# 4. ISSUE 1 (fixed)
The real simulated battery has some voltage. The ADC converts physical voltage to a digital number for the computer to read.
The file src/drivers/adc_driver.c contains the ADC simulation. So essentially there are two different battery values- 
battery_voltage_true and battery_voltage_reported. The difference between these values in this program was very high, which
meant that there was a problem with the ADC conversion. The ADC was shifting the value by 4 bits when creating the raw 
reading, but the scaling function shifted it by 5 bits while turning it back. This is a problem as when the battery reading 
is wrong, safe mode is being enabled when it is not required. So I changed the reverse conversion to shift right by 4, 
matching the original left shift.
## 5. ISSUE 2 (investigated, not fixed)
THERM_STALE: 0 means we are using the current temperature, and when it is 1, we are using an older temperature value instead.
As per the code, when scheduler_bias_flipped becomes true(which is when refresh>=thermal), it deliberately chooses the older 
reading. This is why it's called stale data. The spacecraft makes decisions based on an outdated picture of its temperature.
Also, if two tasks have the same priority, the one that runs fewer times gets selected first. Sensor Buffer Refresh and
Thermal Monitor are both medium priority. This is important as sensor buffer refresh updates the current value, while thermal
monitor reads it. The order in which it is executed can affect whether the thermal monitor uses current value or an older
one. With more time, I would have investigated why scheduler_bias_flipped is being set and the intended order. I would then check whether changing the order removes THERM_STALE.
## 6. ISSUE 3 (investigated, not fixed)
memcpy copies bytes from one memory location to another. Suppose memcpy has to copy 30 bytes but buffer only has room
for 16. The remaining 14 bytes continue past the buffer and may overwrite adjacent memory, including the canary. 
That is a buffer overflow. Canary is a value placed nearby in memory so the program can detect whether something
accidentally overwrote that memory. My investigation process, was at first identifying the warning that I got(Sensor copy
canary modified at tick 16). Then I searched the code for sensor copy canary and temp_guarded_copy- this lead me to the
sensor copy implementation. This is a potential buffer overflow problem. So my preferred path of action would be inspecting the exact destination buffer size and number of bytes passed to memcpy. I would then make sure copy length cannot exceed the destination capacity, rerun the simulation and verify that canary warning at tick 16 does not appear again.
## 7. Fault Injection
My understanding of fault injection is that we are supposed to add a command that pushes the spacecraft into a dangerous
state, predict when it will crash and record that predicted tick.
In src/main.c, I added cmd_set_actuators(100.0,5000.0);
So we intentionally turn the heat and reaction wheel speed way up. Eventually, we reach 85 degrees celcius and vibrations=13.
This causes structural failure at tick 325.
Calculation:
heat added per tick= 1x0.25=0.25
heat lost=0.05
starting temperature=20 C
Therefore, tick 1-20.2 C, tick 2-20.4 C,...
Wheel speed increases by 50RPM per tick
vibrations= 0.5+(5000x0.0025)=13;
Hence spacecraft fails when temp>85C and vibration>12
tick at which failure occurs= (85-20)/0.2=325
