# SMART-OFFICE-
#include <Arduino.h>
#include <WiFi.h>
#include <Wire.h>
#include <Ethernet.h>
#include <SPI.h>
//#include <SparkFun_VL53L1X.h>
#include <IRremoteESP8266.h>
#include <IRsend.h>
#include <VL53L1X.h>
#include <ShiftRegister74HC595.h>


ShiftRegister74HC595<2> sr(5, 17, 16);
IRsend irsend(12);
int khz = 38;
//Optional interrupt and shutdown pins.
#define SHUTDOWN_PIN 2
#define INTERRUPT_PIN 3
// define IR led pin

const IPAddress remote_ip(192, 168, 1, 104);
#define Sensor1_newAddress 47
#define Sensor2_newAddress 45

VL53L1X Sensor1;
VL53L1X Sensor2;

uint16_t distance0, distance1;

#define SHUTDOWN_PIN 2
#define INTERRUPT_PIN 3

#define XSHUT_pin1 32
#define XSHUT_pin2 33
#define dis_Threshold 1300
#define dis_Ignore 60
  
String working_mode = "client";
int people = 0;

// Timeout definition
const unsigned long period = 1000;
unsigned long startMillis, currentMillis;
bool obj = false;

// define ROI centers
uint8_t center[2] = {159, 231};

// object status
enum Object {NOBODY, SOMEONE, NODIR};

// zone status
enum Zone {NONE, OUTSIDE, INSIDE, STABLE};

String mode = "unknown";

enum Object currentStatus = NOBODY;
enum Zone zone;
enum Zone buff = NONE;

enum Zone currentZone(uint16_t distance0, uint16_t distance1){
  if(distance0 < dis_Ignore || distance1 < dis_Ignore){
      return NONE;
  } else if (distance0 < dis_Threshold && distance1 > dis_Threshold){
      return OUTSIDE;
  } else if (distance1 < dis_Threshold && distance0 > dis_Threshold){
      return INSIDE;
  } else if (distance0 < dis_Threshold && distance1 < dis_Threshold){
      return STABLE;
  } else {return NONE;}
}

void lightIR(String lightmode){

  // light IR code
  uint16_t light_on[68] = {9024,4512,564,564,564,564,564,564,564,564,564,564,564,1692,564,564,564,564,564,564,564,564,564,564,564,
                          564,564,564,564,564,564,564,564,1692,564,1692,564,1692,564,564,564,564,564,1692,564,564,564,564,564,564,
                          564,564,564,564,564,1692,564,1692,564,564,564,1692,564,1692,564,1692,564,46524};

  uint16_t light_off[68] = {9024,4512,564,1692,564,1692,564,1692,564,564,564,564,564,564,564,1692,564,564,564,1692,564,1692,564,1692,564,564,
                            564,1692,564,564,564,1692,564,1692,564,1692,564,564,564,564,564,1692,564,564,564,564,564,564,564,1692,564,1692,564,
                            564,564,1692,564,1692,564,1692,564,564,564,564,564,564,564,564};

  if (lightmode == "on"){
      irsend.sendRaw(light_on, 68, khz);
      delay(500);
      irsend.sendRaw(light_on, 68, khz);
      ESP_LOGE("Light", "Turned on");
  } else if (lightmode == "off"){
      irsend.sendRaw(light_off, 68, khz);
      delay(500);
      irsend.sendRaw(light_off, 68, khz);
      ESP_LOGE("Light", "Turned off");
  }
}

void lightController(int people, String currentMode){
  if (people == 0 && currentMode == "on"){
      mode = "off";
      lightIR(mode);
  } else if (people > 0 && currentMode == "off"){
      mode = "on";
      lightIR(mode);
  } else if (people == 0 && currentMode == "unknown"){
      mode = "off";
      lightIR(mode);
  } else if (people > 0 && currentMode == "unknown"){
      mode = "on";
      lightIR(mode);
  }
}

void acIr(String mode){
  
  // AC IR code
  uint16_t ac_on[199] = {4472, 4358,  580, 1604,  580, 516,  578, 1584,  606, 1580,  606, 516,  580, 516,  578, 1580,  606, 518,  580, 
          520,  578, 1584,  606, 518,  578, 518,  578, 1580,  606, 1580,  604, 518,  578, 1588,  602, 1588,  602, 518,  576, 520,  
          576, 1608,  576, 1610,  574, 1610,  574, 1610,  574, 1612,  572, 526,  572, 1612,  550, 1636,  548, 548,  548, 548,  548, 
          548,  548, 548,  548, 550,  548, 550,  548, 548,  548, 1636,  548, 548,  548, 548,  548, 546,  550, 546,  548, 550,  548, 
          1638,  548, 1636,  548, 548,  548, 1642,  548, 1636,  548, 1636,  548, 1636,  548, 1638,  550, 5230,  4414, 4392,  548, 
          1638,  548, 548,  548, 1640,  548, 1638,  546, 550,  548, 548,  574, 1612,  546, 552,  572, 526,  548, 1640,  574, 524,  
          572, 524,  572, 1612,  548, 1638,  572, 524,  572, 1618,  572, 1618,  548, 548,  572, 526,  546, 1638,  572, 1612,  572, 
          1614,  572, 1614,  572, 1614,  572, 526,  546, 1638,  548, 1638,  572, 524,  546, 550,  570, 524,  572, 524,  572, 526,  
          572, 526,  572, 526,  546, 1638,  572, 524,  572, 524,  548, 550,  572, 524,  572, 526,  572, 1616,  570, 1614,  570, 526,  
          572, 1618,  572, 1614,  572, 1614,  570, 1614,  546, 1642,  546};

  uint16_t ac_off[99] = {4350,4350, 550,1600, 600,500, 550,1600, 600,1600, 550,500, 600,500, 550,1600, 600,500, 550,500, 600,1600, 550,
          550, 550,550, 550,1550, 600,1600, 550,500, 600,1600, 550,550, 550,1600, 550,1600, 600,1600, 550,1550, 600,550, 550,1600, 550,
          1600, 600,1550, 600,500, 600,500, 550,500, 600,500, 600,1550, 600,500, 600,500, 550,1650, 550,1550, 600,1600, 550,550, 550,550, 
          550,500, 550,550, 550,550, 500,550, 550,550, 550,500, 550,1650, 550,1600, 550,1650, 500,1650, 550,1600, 550};

  if (mode == "on"){
      irsend.sendRaw(ac_on, 199, khz);
      ESP_LOGE("AC", "Turned on........");
  } else if (mode == "off"){
      irsend.sendRaw(ac_off, 99, khz);
      ESP_LOGE("AC", "Turned off.......");
  }
}

void standaloneMode(int people, String currentMode){
  lightController(people, currentMode);
  if (people == 0 && currentMode == "on"){
      mode = "off";
      acIr(mode);
  } else if (people > 0 && currentMode == "off"){
      mode = "on";
      acIr(mode);
  } else if (people == 0 && currentMode == "unknown"){
      mode = "off";
      acIr(mode);
  } else if (people > 0 && currentMode == "unknown"){
      mode = "on";
      acIr(mode);
  }
}

// uint16_t readDistance(SFEVL53L1X distanceSensor, int a){
//     distanceSensor.startRanging(); //Write configuration bytes to initiate measurement
//     while (!distanceSensor.checkForDataReady())
//     {
//         ESP_LOGE("VL53L1X custom sensor", ".......");
//         delay(1);
//     }
//     uint16_t distance = distanceSensor.getDistance(); //Get the result of the measurement from the sensor
//     distanceSensor.clearInterrupt();
//     distanceSensor.stopRanging();
//     return distance;
//   }

uint16_t readDistance(VL53L1X distanceSensor){
  distanceSensor.startContinuous(15);
  distanceSensor.read();
  distanceSensor.stopContinuous();
  return distanceSensor.ranging_data.range_mm;
}

int value, digit1, digit2, digit3, digit4;

uint8_t countB[10] = {
  B11000000, //0
  B11111001, //1
  B10100100, //2
  B10110000, //3
  B10011001, //4
  B10010010, //5
  B10000011, //6
  B11111000, //7
  B10000000, //8
  B10011000  //9
};

void dislay(int people){
  //Split count to digits:
  digit2 = people % 10;
  digit1 = (people / 10) % 10;
  //Send them to 7 segment displays
  uint8_t countToPrint[] = {countB[digit1], countB[digit2]};
  sr.setAll(countToPrint);
  //Reset them for next time
  digit1 = 0;
  digit2 = 0;
  delay(50); // Repeat every 1 sec
}

void setup(void)
{
//    pinMode(XSHUT_pin1, OUTPUT);
//    pinMode(XSHUT_pin2, OUTPUT);
    // This will be called by App.setup()
    Wire.begin();
    Wire.setClock(400000); // use 400 kHz I2C
    Serial.begin(115200);
    Serial.println("Begin.....");

    pinMode(XSHUT_pin1, OUTPUT);
    pinMode(XSHUT_pin2, OUTPUT);
      
    Wire.begin();
    Wire.setClock(100000); 
    
    
    pinMode(XSHUT_pin1, INPUT);
    if (!Sensor1.init())
    {
        Serial.println("VL53L1X custom sensor, Failed to detect and initialize sensor1!");
        while (1);
    }
    delay(10);
    Sensor1.setAddress(Sensor1_newAddress);
    
    pinMode(XSHUT_pin2, INPUT);
    if (!Sensor2.init())
    {
        Serial.println("VL53L1X custom sensor, Failed to detect and initialize sensor2!");
        while (1);
    }
    delay(10);
    Sensor2.setAddress(Sensor2_newAddress);
    
      
    Sensor1.setTimeout(500);
    Sensor2.setTimeout(500);

    Sensor1.setDistanceMode(VL53L1X::Medium);
    Sensor1.setMeasurementTimingBudget(15000);
    Sensor2.setDistanceMode(VL53L1X::Medium);
    Sensor2.setMeasurementTimingBudget(15000);
    irsend.begin();
}

void loop(void)
{
    distance0 = readDistance(Sensor1);
    distance1 = readDistance(Sensor2);
    Serial.print("Distance 0: ");
    Serial.println(distance0);
    Serial.print("Distance 1: ");
    Serial.println(distance1);

    // TODO: check connection with server

    enum Zone current_zone = currentZone(distance0, distance1);
    
    if((buff == INSIDE || buff == OUTSIDE) && current_zone == NONE){
        currentStatus == NOBODY;
        buff = NONE;
    }

    if (currentStatus == SOMEONE && currentStatus != STABLE){
      if(current_zone == NONE){
        currentStatus = NOBODY;
        buff = NONE;
      }
      if ((buff == OUTSIDE) && (current_zone == INSIDE))
      {
        people++;
        currentStatus == NOBODY;
        buff = NONE;
      } else if ((buff == INSIDE) && (current_zone == OUTSIDE))
      {
        if(people == 0){
            ESP_LOGD("ISSUE", "Someone went out but current people is 0, can't count down");
        } else {
            people--;
        }
        // people--;
        currentStatus == NOBODY;
        buff = NONE;
      }
    } else if(currentStatus == NOBODY && (current_zone == INSIDE || current_zone == OUTSIDE)){
      ESP_LOGD("Status","Detected someone");
      buff = current_zone;
      currentStatus = SOMEONE;
    } else if(currentStatus == NOBODY && current_zone == STABLE){
      currentStatus = NODIR;
      buff = STABLE;
      ESP_LOGD("Status","Detected someone but didn't know direction");
    } else if(currentStatus == NODIR){
      if (current_zone == INSIDE){
        people++;
        buff = NONE;
        currentStatus = NOBODY;
      } else if(current_zone == OUTSIDE){
        if(people == 0){
            ESP_LOGD("ISSUE", "Someone went out but current people is 0, can't count down");
        } else {
            people--;
        }
        // people--;
        buff = NONE;
        currentStatus = NOBODY;
      } else if(current_zone == NONE){
        buff = NONE;
        currentStatus = NOBODY;
      }
    }
    dislay(people);
    Serial.print("people: ");
    Serial.println(people);
    lightController(people, mode);

    // control air conditioner and light using IR if currently in standalone mode
    if (working_mode == "standalone"){
        standaloneMode(people, mode);
    }
    return;
}
