#include <HardwareSerial.h>

HardwareSerial GSM(2); // Assuming you're using UART2 of ESP32
const char phone_no[] = "+60198096517"; // Change with phone number to call
#define SMOKE_SENSOR_PIN 15 // Assuming you're using GPIO pin 15 for the smoke sensor
#define SIM800L_RESET_PIN 4 // Assuming you're using GPIO pin 4 for SIM800L reset
#define RESET_BUTTON_PIN 14 // Assuming you're using GPIO pin 14 for the reset button
#define GREEN_LED_PIN 25 // Assuming you're using GPIO pin 25 for the green LED
#define RED_LED_PIN 26 // Assuming you're using GPIO pin 26 for the red LED
#define BUZZER_PIN 19 // Assuming you're using GPIO pin 19 for the buzzer

bool callInProgress = false;
unsigned long smokeDetectedTime = 0;

void setup() {
  Serial.begin(9600);
  GSM.begin(9600, SERIAL_8N1, /*Rx Pin*/ 16, /*Tx Pin*/ 17); // Assuming Rx is GPIO 16 and Tx is GPIO 17

  pinMode(SMOKE_SENSOR_PIN, INPUT); // Set smoke sensor pin as input
  pinMode(SIM800L_RESET_PIN, INPUT_PULLUP); // Set SIM800L reset pin as input with internal pull-up resistor
  pinMode(RESET_BUTTON_PIN, INPUT_PULLUP); // Set reset button pin as input with internal pull-up resistor
  pinMode(GREEN_LED_PIN, OUTPUT); // Set green LED pin as output
  pinMode(RED_LED_PIN, OUTPUT); // Set red LED pin as output
  pinMode(BUZZER_PIN, OUTPUT); // Set buzzer pin as output

  Serial.println("Initializing...");
  initModule("AT", "OK", 1000); // Handshake test

  // System is ready, turn on green LED
  digitalWrite(GREEN_LED_PIN, HIGH);

  // Test buzzer pin output
  digitalWrite(BUZZER_PIN, HIGH);
}

void loop() {
  int smokeLevel = analogRead(SMOKE_SENSOR_PIN);
  Serial.println(smokeLevel);

  if (smokeLevel > 100) {
    // If smoke level is above threshold, check for duration
    if (smokeDetectedTime == 0) {
      // If first time above threshold, record time
      smokeDetectedTime = millis();
    } else {
      // If above threshold for 2 seconds, make call
      if (millis() - smokeDetectedTime >= 2000 && !callInProgress) {
        flashRedLED(); // Flash the red LED when smoke is detected and calling
        callUp(phone_no);
      }
    }
  } else {
    // Reset smoke detected time if below threshold
    smokeDetectedTime = 0;
  }

  // Check if call is in progress
  if (callInProgress && GSM.available()) {
    if (GSM.find("+COLP")) {
      Serial.println("NOW MAKING EMERGENCY CALLING");
      callInProgress = false;
      activateBuzzer(); // Activate the buzzer after making the call
    }
  }

  // Check if alarm is active and smoke is detected
  if (callInProgress && smokeLevel > 100) {
    activateBuzzer(); // Activate the buzzer when both conditions are met
  } else {
    deactivateBuzzer(); // Deactivate the buzzer otherwise
  }

  // Check if SIM800L reset button is pressed
  if (digitalRead(SIM800L_RESET_PIN) == LOW) {
    Serial.println("Resetting SMOKE SPOTTER");
    delay(1000); // Wait for a brief moment to ensure serial output is flushed
    ESP.restart(); // Reset the ESP32
  }

  // Check if reset button is pressed
  if (digitalRead(RESET_BUTTON_PIN) == LOW) {
    Serial.println("Resetting system...");
    delay(1000); // Wait for a brief moment to ensure stability
    ESP.restart(); // Reset the ESP32
  }
  
  delay(100);
}

void callUp(const char *number) {
  Serial.println("Making a call...");
  GSM.print("ATD "); GSM.print(number); GSM.println(";");
  callInProgress = true;
}

void initModule(const String &cmd, const char *res, int t) {
  while (true) {
    Serial.println(cmd);
    GSM.println(cmd);
    delay(100);
    
    unsigned long startTime = millis();
    while (millis() - startTime < t) {
      if (GSM.find(res)) {
        Serial.println(res);
        delay(t);
        return;
      }
    }
    Serial.println("Error: Module not responding as expected.");
    delay(t); // Retry after a delay
  }
}
