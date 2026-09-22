# Gesture-controlled Timer
**Project Type:** Embedded Systems Demonstration

**Status:** Completed

**Date:** September 2026



## Table of Contents
- [Overview](#overview)
- [Components](#components)
- [How It Works](#how-it-works)
- [Assets](#assets)
  - [Breadboard Diagram](#breadboard-diagram)
  - [Circuit Diagram](#circuit-diagram)
  - [Demonstration](#demonstration)
  - [Source Code](#source-code)
- [Key Takeaways](#key-takeaways)


  
## Overview
A responsive electronic timer that utilizes specific input patterns from an ultrasonic sensor to toggle functions instead of traditional buttons.


## Components
* Arduino Uno
* HC-SR04 Ultrasonic Distance Sensor
* 16×2 I2C LCD
* Passive Buzzer
* 2 × 220Ω Resistors
* Push Button
* LED


  
## How It Works

The system operates as a timer controlled primarily through hand gestures detected by an HC-SR04 ultrasonic sensor. A latching push button serves as the manual power button and is the only way to toggle it on. Once powered on and unpaused, the timer starts from 00:00 and counts up to 59:59, with the elapsed time displayed on the LCD.

The HC-SR04 ultrasonic sensor continuously measures the distance of objects in front of it. The detection range is set to approximately 3-18 cm from the sensor. Different functions trigger depending on how long the hand remains within this range. A short hold of 0.25s pauses or resumes the timer, a 1.25s hold resets it, and a 2.25s hold switches the system off.

A stage variable maintains the state of the most recently executed action while a gesture is actively being made. Once the conditions for an action are met, the program advances to the corresponding stage, preventing the same action from being repeatedly executed and allowing it to progress to subsequent actions if the gesture is maintained. The stage resets when the gesture ends. The buzzer provides audible feedback whenever an action is registered.



## Assets
### Breadboard Diagram
![Breadboard Diagram](<Assets/Breadboard Diagram.png>)

### Circuit Diagram
![Circuit Diagram](<Assets/Circuit Diagram.png>)

### Demonstration
![Image](<Assets/Image.png>)

[Video](<Assets/Demonstration.mp4>)

### Source Code
[Arduino Sketch](<Assets/GestureTimer/GestureTimer.ino>)
```cpp
#include <Wire.h> 
#include <LiquidCrystal_I2C.h>
#include <LCDGraph.h>


int buttonVolt = 0;
int currentState = LOW;
int lastState = LOW;
long lastDebounceTime = 0;
const long DEBOUNCE_DELAY = 50;

#define BUTTON 8
#define LED 7

#define TRIG 3
#define ECHO 2

#define BUZZER 12

#define GESTURE_PAUSE 250    
#define GESTURE_RESET 1250  
#define GESTURE_OFF   2250  

float distanceHC;
long durationHC;
unsigned long lastTimeHC = 0;
const long readDelayHC = 100;


LiquidCrystal_I2C lcd(0x27, 16, 2);
unsigned long previousMillis = 0;
const long SECOND = 1000;
int seconds = 0;
int minutes = 0;


bool readingGesture = false;

const int detectDistance = 18;

void runTimer();
void updateDisplay();
void powerOn();
void getDistance();
void powerToggle();
void powerOff();
void PowerOn();
void processGesture();
void executeGesture();
void beep();

bool timerRunning = false;


void setup() {
  Wire.begin();
  Wire.setClock(400000L);
  Serial.begin(9600);
  Serial.setTimeout(50);
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(BUTTON, INPUT_PULLUP); 
  pinMode(LED, OUTPUT);
  pinMode(BUZZER, OUTPUT);
}

unsigned long currentMillis = millis();




void loop() {
  powerToggle();
  currentMillis = millis();

  if (currentMillis - previousMillis >= SECOND) {
    previousMillis = currentMillis;
    runTimer();
    updateDisplay();
  
  }
  getDistance();
  processGesture();
  executeGesture();

 
}


void powerToggle() {
  int reading = digitalRead(BUTTON);

  if (reading != lastState) {
    lastDebounceTime = millis();
  }

  if ((millis() - lastDebounceTime) > DEBOUNCE_DELAY) {
    if (reading != buttonVolt) {
      buttonVolt = reading;
      if (buttonVolt == LOW) {          
        if (currentState == LOW) {
          powerOn();
        } else {
          powerOff();
        }
      }
    }
  }
  lastState = reading;
}


void powerOn() {
  currentState = HIGH;
  timerRunning = false;
  digitalWrite(LED, HIGH);
  
  seconds = 0;
  minutes = 0;
  previousMillis = millis();   
  lcd.init();
  lcd.display();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("AWAITING GESTURE");
  beep();

  readingGesture = false;
}

void powerOff() {
  currentState = LOW;
  timerRunning = false;
  digitalWrite(LED, LOW);
  lcd.noBacklight();
  lcd.noDisplay();
  readingGesture = false;
  beep();

}

void runTimer(){
  if (timerRunning == true){
    seconds++;
    if (seconds >= 60) {
        seconds = 0;
        minutes++;
        if (minutes >= 60) {
          minutes = 0; 
        }
    }
  }  
}

void updateDisplay(){

  lcd.setCursor(6, 1);  
  if (minutes < 10) lcd.print("0");
  lcd.print(minutes);
  lcd.print(":");
  
  if (seconds < 10) lcd.print("0");
  lcd.print(seconds);

}

void getDistance(){

  if (currentState == HIGH){
    digitalWrite(TRIG, LOW);
    unsigned long currentTimeHC = millis();
    if (currentTimeHC - lastTimeHC >= readDelayHC) {
      lastTimeHC = currentTimeHC;

      digitalWrite(TRIG, LOW);
      delayMicroseconds(2);

      digitalWrite(TRIG, HIGH);
      delayMicroseconds(10);
      digitalWrite(TRIG, LOW);

      durationHC = pulseIn(ECHO, HIGH, 7500);
      distanceHC = (durationHC * 0.0343)/2;

      if (distanceHC >= 35 || distanceHC == 0) {
        distanceHC = 0;
      }
      else{
        //Serial.println(distanceHC);
 
      }
    }
  }
}

void processGesture(){
  static unsigned long readThreshold = 0;
  static unsigned long disengageThreshold = 0;
  
  if (distanceHC >= 3 && distanceHC <= detectDistance){
    if (currentMillis - readThreshold >= 300) {
      readThreshold = currentMillis;
      readingGesture = true;
      lcd.setCursor(0, 0);
      lcd.print("READING  GESTURE");
    }
  }

  if (distanceHC < 3 || distanceHC > detectDistance){
    if (currentMillis - disengageThreshold >= 600) {
      disengageThreshold = currentMillis;
      if (distanceHC < 3 || distanceHC > detectDistance){
        readingGesture = false;
      lcd.setCursor(0, 0);
      lcd.print("AWAITING GESTURE");
      }
    }      
  }
}


void executeGesture() {
  static unsigned long holdStart = 0;
  static uint8_t stage = 0;

  if (!readingGesture) {
    holdStart = currentMillis; 
    stage = 0;
    return;
  }

  unsigned long held = currentMillis - holdStart;

  if (stage < 1 && held >= GESTURE_PAUSE){
    stage = 1;
    beep();
    timerRunning = !timerRunning;

  } else if (stage < 2 && held >= GESTURE_RESET){
    stage = 2;
    beep();
    char time[9];
    sprintf(time, "%02d:%02d", minutes, seconds);
    Serial.println(time);
    seconds = 0;
    minutes = 0;
    timerRunning = false;

  } else if (stage < 3 && held >= GESTURE_OFF){
    stage = 3;
    beep();
    powerOff();
  }
}

void beep(){
  digitalWrite(BUZZER, HIGH); 
  delay(100);
  digitalWrite(BUZZER, LOW);
} 
```

## Key Takeaways
* Functions and state management are key to structuring larger programs for more complex systems.
* Managing event timing and utilizing non-blocking delays increase the efficiency and responsiveness of the system.
* Integrating multiple components might lead to interference between some of their operations. This has to be accounted for.
* Sensors and programmed logic can be combined to form an alternative method of acquiring user input.
* Filtering sensor readings helps reduce the effect of noise and prevents unintended system responses.
