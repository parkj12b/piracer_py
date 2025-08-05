#include <Canbus.h>
#include <defaults.h>
#include <global.h>
#include <mcp2515.h>
#include <mcp2515_defs.h>

#define SENSOR_PIN 2
#define SAMPLE_PERIOD 500 //0.5sec

const int HOLES_PER_REV = 20;
const float WHEEL_DIAMETER_M = 0.067;
const float WHEEL_RADIUS_CM = (WHEEL_DIAMETER_M / 2.0) * 100.0;

int pulse_count = 0;
long last_sample_time = 0;

void countPulse() {
  pulse_count++;
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  if (Canbus.init(CANSPEED_500)) {
    Serial.println("CAN Init ok");
  }
  else {
    Serial.println("CAN Init failed");
  }

  pinMode(SENSOR_PIN, INPUT);
  attachInterrupt(digitalPinToInterrupt(SENSOR_PIN), countPulse, RISING);

  last_sample_time = millis();
}

void loop() {
  long now = millis();
  if (now - last_sample_time >= SAMPLE_PERIOD) {
    detachInterrupt(digitalPinToInterrupt(SENSOR_PIN));

    float rev_per_sec = (pulse_count / (float)HOLES_PER_REV) / (SAMPLE_PERIOD / 1000.0);
    float speed = rev_per_sec * 2.0 * 3.141592 * WHEEL_RADIUS_CM;
    float rpm = rev_per_sec * 60.0;

    Serial.print("Pulses: ");
    Serial.print(pulse_count);
    Serial.print(" | Speed: ");
    Serial.print(speed);
    Serial.print(" cm/s | RPM: ");
    Serial.println(rpm);

    int speed_int = (int)speed;
    int speed_frac = round((speed - speed_int) * 100);
    int rpm_int = (int)rpm;
    int rpm_frac = round((rpm - rpm_int) * 100);

    tCAN message;
    message.id = 0x123;
    message.header.rtr = 0;
    message.header.length = 6;
    message.data[0] = (speed_int >> 8) & 0xFF;
    message.data[1] = speed_int & 0xFF;
    message.data[2] = speed_frac;
    message.data[3] = (rpm_int >> 8) & 0xFF;
    message.data[4] = rpm_int & 0xFF;
    message.data[5] = rpm_frac;

    mcp2515_bit_modify(CANCTRL, (1<<REQOP2)|(1<<REQOP1)|(1<<REQOP0), 0);
    mcp2515_send_message(&message);

    pulse_count = 0;
    last_sample_time = now;
    
    attachInterrupt(digitalPinToInterrupt(SENSOR_PIN), countPulse, FALLING);
  }
}
