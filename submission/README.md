## Code Level Work

## class ShanWanGamepad

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


## read_data()
### gamepad.py (piracer library)

```
def read_data(self) -> ShanWanGamepadInput:
	_, button_number, button_state, _, axis_number, axis_val = super(ShanWanGamepad, self).poll()
```

### 