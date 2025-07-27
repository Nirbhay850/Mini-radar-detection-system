# Mini-radar-detection-system
An Arduino-based radar system using ultrasonic sensor, servo motor, LCD, and laser module for object detection
i have used the following components : 
Ultrasonic sensor (HC-SR04)
- Servo motor (SG90)
- I2C LCD display
- Laser module (KY-008)
- i 3d printed the base, the arm, the plane and the truck

  how it works :
  - Servo motor sweeps the ultrasonic sensor across angles.
  - When an object is detected under 10 cm:
  - Laser blinks
  - LCD displays distance and bar graph
  - Servo pauses for 1s, then ignores detection for 2s

This is the code : 
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

// Objects
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo myServo;

// Pins
const int trigPin = 9;
const int echoPin = 10;
const int laserPin = 7;
const int servoPin = 6;

// Timing
unsigned long lastLcdUpdate = 0;
const unsigned long lcdRefreshDelay = 500;
unsigned long ignoreStartTime = 0;
bool ignoringObject = false;
bool justDetected = false;

// Servo tracking
int currentServoAngle = 0;
int servoDirection = 1; // 1 = forward, -1 = backward

void setup() {
  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(laserPin, OUTPUT);
  digitalWrite(laserPin, LOW);

  lcd.init();
  lcd.backlight();

  myServo.attach(servoPin);
  myServo.write(currentServoAngle);

  Serial.begin(9600);
}

void loop() {
  float distance = getDistance();
  unsigned long now = millis();
  bool objectDetected = (distance > 0 && distance < 10.0);

  Serial.print("Distance: ");
  Serial.println(distance);

  if (objectDetected && !ignoringObject) {
    justDetected = true;
    ignoringObject = true;
    ignoreStartTime = now;

    // Blink laser 3 times
    for (int i = 0; i < 3; i++) {
      digitalWrite(laserPin, HIGH);
      delay(100);
      digitalWrite(laserPin, LOW);
      delay(100);
    }

    // Stop servo for 1 second
    delay(1000);
  }

  // End ignore mode after 2 seconds
  if (ignoringObject && (now - ignoreStartTime >= 2000)) {
    ignoringObject = false;
  }

  // LCD update
  if (now - lastLcdUpdate >= lcdRefreshDelay) {
    lcd.clear();
    lcd.setCursor(0, 0);

    if (objectDetected) {
      lcd.print("Object: ");
      lcd.print(distance, 1);
      lcd.print(" cm");

      int bars = map(distance, 1, 10, 16, 1);
      lcd.setCursor(0, 1);
      for (int i = 0; i < 16; i++) {
        if (i < bars) lcd.write(255);
        else lcd.print(" ");
      }
    } else {
      lcd.print("No object <10cm");
      lcd.setCursor(0, 1);
      lcd.print("                ");
    }

    lastLcdUpdate = now;
  }

  // Move servo (skip one cycle when just detected)
  if (!justDetected) {
    currentServoAngle += servoDirection;

    if (currentServoAngle >= 180) {
      currentServoAngle = 180;
      servoDirection = -1;
    } else if (currentServoAngle <= 0) {
      currentServoAngle = 0;
      servoDirection = 1;
    }

    myServo.write(currentServoAngle);
    delay(10);
  } else {
    justDetected = false; // allow servo to move again next loop
  }

  delay(50); // main loop delay
}

float getDistance() {
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  long duration = pulseIn(echoPin, HIGH, 30000);  // 30ms timeout
  if (duration == 0) return 999;
  return duration * 0.034 / 2;
}






