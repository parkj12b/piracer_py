## Sequence Diagram
```
[Gamepad Input]
   ↓
   (ShanWanGamepad or Xbox 2.4GHz receiver)
   ↓   USB/Bluetooth
┌──────────────────────┐
│ Raspberry Pi / Host  │
│  - Reads gamepad     │
│  - Maps throttle      │
│  - Sends PWM duty     │
│    via I2C            │
└─────────┬────────────┘
          ↓
       I2C Bus
          ↓
┌────────────────────────────┐
│ PCA9685 PWM Driver Chip    │
│  - Receives I2C command    │
│  - Stores PWM duty in regs │
│  - Outputs PWM signal      │
│    on selected channel     │
└─────────┬────────────┬─────┘
          │            │
          │            │
          ▼            ▼
     [AIN1/2]      [PWMA/PWMB]
  (GPIO or PWM)     (PWM Signal)
          \            /
           ▼          ▼
       ┌────────────────────┐
       │ TB6612FNG Motor    │
       │ Driver (H-Bridge)  │
       │  - Receives dir+PWM│
       │  - Switches power  │
       │  - Drives motor    │
       └─────┬──────────────┘
             ▼
         [Motor Terminals]
           A01 ↔ A02
             ↓
         🌀 Motor Spins!

```

## Overview
### Controller Input

1. 2.4ghz signal from controller to Pi
2. USB HID protocol transmits joystick/button state
3. Linux USB driver ensures proper input reading
4. Python reads continuously from /dev/input/js0
    - jsX is the joystick device
5. Joystick axis → normalized
6. Donkey Car scripts converts joystick to throttle
    - python code by the programmar

### Output to Brushed DC Motor

1. Sends throttle to motor control interface
2. Maps duty cycle for PWM
3. Continuously update PWM output
4. GPIO/PWM driver ensures consistent duty cycle output
5. Optional I2C/SPI if using a motor controller chip
    1. PCA9685
6. PWM signal via PCB circuit → H-bridge → motors spins at % power

## Hardware Faults
- Since all SEA:ME teams share the same 2.4 GHz channel, controllers can sometimes interfere with each other.

# Code Level Workings
## Controller Input
### class ShanWanGamepad

Class ShanWanGamepad -> Class Joystick
- class ShanWan Gamepad inherits class joystick

### class Joystick init

```
# defaults device to /dev/input/js0. 
# joystick uses naming scheme jsX


def __init__(self, dev_fn='/dev/input/js0') -> None:
	self.axis_states = {}
	self.button_states = {}
	self.axis_names = {}
	self.button_names = {}
	self.axis_map = []
	self.button_map = []
	self.jsdev = None
	self.dev_fn = dev_fn
```

### button_pushed.py

```
while True:
	gamepad_input = shanwan_gamepad.read_data()

	""" 
		depending on the what gets returned from gamepad.read_data()
		change piracer throttle and steering
	"""
	steering = -gamepad_input.analog_stick_left.x
        if gamepad_input.button_a == 1:
            throttle = gamepad_input.button_a * 0.5
        else:
            throttle = -gamepad_input.button_b * 0.5

        print(f'throttle ={throttle}, steering={steering}')

        piracer.set_throttle_percent(throttle)
        piracer.set_steering_percent(steering)

```

### read_data() - gamepad.py

```
def read_data(self) -> ShanWanGamepadInput:
	_, button_number, button_state, _, axis_number, axis_val = super(ShanWanGamepad, self).poll()

	if axis_number == 0:
		self.gamepad_input.analog_stick_left.x = axis_val
	elif axis_number == 1:
		self.gamepad_input.analog_stick_left.y = -axis_val
	elif button_number == 10:
		self.gamepad_input.analog_stick_left.z = button_state
	...
```
-  read_data uses super().poll() / joystick.poll() internally

### poll()
```
def poll(self) -> Tuple[Optional[str], Optional[int], Optional[bool], Optional[str], Optional[int], Optional[float]]:
	evbuf = self.jsdev.read(8)
```
- performs blocking read operation
- blocks until device returns any input (sent by the controller)
- values are sent in byte stream

```
	tval, value, typev, number = struct.unpack('IhBB', evbuf)
```
- unpack the value into struct and continues to return meaningful value

## Output to Brushed DC Motor
### rc_example.py
```
	piracer.set_throttle_percent(throttle)
	piracer.set_steering_percent(steering)
```

### set_throttle_percent()
```
def set_steering_percent(self, value: float) -> None:
	"""Set the desired steering value in percent.

	:param float value: Steering angle in percent (left: 0.0 to -1.0, right: 0.0 to +1.0, neutral: 0.0)
	"""
	self._set_channel_active_time(self._get_50hz_duty_cycle_from_percent(-value), self.steering_pwm_controller,
									self.PWM_STEERING_CHANNEL)
	pass
```

### class PiracerBase
```
class PiRacerBase:

def __init__(self) -> None:
	self.i2c_bus = busio.I2C(SCL, SDA)
	self.display = SSD1306_I2C(128, 32, self.i2c_bus, addr=0x3c)
```

- initialize the abstraction of I2c_bus and display SSD1306_I2C

### class PiRacerStandard
```
class PiRacerStandard(PiRacerBase):
def __init__(self) -> None:
    super().__init__()
    # initializes PCA9685 at address 0x40 of self.i2c_bus
    self.steering_pwm_controller = PCA9685(self.i2c_bus, address=0x40)
    self.steering_pwm_controller.frequency = self.PWM_FREQ_50HZ

    self.throttle_pwm_controller = PCA9685(self.i2c_bus, address=0x60)
    self.throttle_pwm_controller.frequency = self.PWM_FREQ_50HZ

    self.battery_monitor = INA219(self.i2c_bus, addr=0x41)

    self._warmup()
```

- Initializes PCA9685
    - 2x motor controller unit one for steering and one for throttle
- INA219
    - https://www.ti.com/lit/ds/symlink/ina219.pdf?ts=1753225093284

### PCA9685

- https://cdn-shop.adafruit.com/datasheets/PCA9685.pdf
- PCA9685 is a 16-channel, 12-bit PWM controller that communicates over **I2C**.
    - 16 pin controlls
    - 12-bit resolution = 4096 duty cycle
- Instead of a internal loop it uses:
    - A fixed internal PWM generator (oscillator + counter)
    - It continuously read values from its internal registers (not memory with logic)
    - It then generates PWM output based on those register values
- PWM keeps running even if you don’t send new commands
    
    ![image.png](attachment:755aa386-e9ea-43cc-aa81-f76525f0f66a:image.png)
    
    ```python
    # Registers:
        mode1_reg = UnaryStruct(0x00, "<B")
        mode2_reg = UnaryStruct(0x01, "<B")
        prescale_reg = UnaryStruct(0xFE, "<B")
        pwm_regs = StructArray(0x06, "<HH", 16)
    ```
    
    - strictArray defined multiple struct in array in this case 16 to represent 16 channels

### chip to motor

- PCA9685 generates digital PWM signal
    - PCA9685 is a logic level chip and cannot driver motors directly (3.3V or 5V, low current)
- signal goes to Motor Driver (e.g. H-Bridge)
- TB6612FNG is the chip on piracer
    - https://cdn.sparkfun.com/assets/0/1/b/b/3/TB6612FNG.pdf
        
        
        | Pin | Function |
        | --- | --- |
        | `PWMA` | PWM input → **controls speed** |
        | `AIN1` | Direction input 1 |
        | `AIN2` | Direction input 2 |
        | `A01`, `A02` | Output to Motor A terminals |
