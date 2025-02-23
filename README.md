//atay
//ggs

#include <Arduino.h>
#include <LiquidCrystal.h>
#include <EEPROM.h>
#include <Servo.h>
#include <SoftwareSerial.h>

#define RS 1
#define EN 2
#define D4 3
#define D5 4
#define D6 5
#define D7 6

LiquidCrystal lcd(RS, EN, D4, D5, D6, D7);

#define NEEDS_BUTTON_PIN 7
#define EMERGENCY_BUTTON_PIN 8

#define SERVO_PIN 9

#define rxPin 10
#define txPin 11
SoftwareSerial sim800(rxPin, txPin);

#define RELAY_1 22
#define RELAY_2 24

int i = 0;
int impulsCount = 0;
float total_amount = 0;
float needsTotal = 0;
float emergencyTotal = 0;

Servo moneyServo;

bool isNeedsSelected = false;
bool isEmergencySelected = false;

String smsStatus, senderNumber, receivedDate, msg;
boolean isReply = false;

const String PHONE = "ENTER_PHONE_HERE";

void setup() {
  Serial.begin(9600);
  sim800.begin(9600);
  
  lcd.begin(16, 2);
  lcd.setCursor(0, 0);
  lcd.print("Total Amount:");
  
  attachInterrupt(26, incomingImpuls, FALLING);

  EEPROM.get(0, total_amount);
  EEPROM.get(4, needsTotal);
  EEPROM.get(8, emergencyTotal);

  pinMode(NEEDS_BUTTON_PIN, INPUT_PULLUP);
  pinMode(EMERGENCY_BUTTON_PIN, INPUT_PULLUP);

  moneyServo.attach(SERVO_PIN);
  moneyServo.write(0);

  pinMode(RELAY_1, OUTPUT);
  pinMode(RELAY_2, OUTPUT);
  digitalWrite(RELAY_1, HIGH);
  digitalWrite(RELAY_2, HIGH);

  sim800.print("AT+CMGF=1\r");
  delay(1000);
  sim800.println("AT+CMGD=1,4");
  delay(1000);
  sim800.println("AT+CMGDA= \"DEL ALL\"");
  delay(1000);
}

void incomingImpuls() {
  impulsCount = impulsCount + 1;
  i = 0;
}

void loop() {
  while(sim800.available()){
    parseData(sim800.readString());
  }

  while(Serial.available()) {
    sim800.println(Serial.readString());
  }

  if (digitalRead(NEEDS_BUTTON_PIN) == LOW) {
    isNeedsSelected = true;
    isEmergencySelected = false;
    moneyServo.write(0);
    delay(200);
  } else if (digitalRead(EMERGENCY_BUTTON_PIN) == LOW) {
    isEmergencySelected = true;
    isNeedsSelected = false;
    moneyServo.write(90);
    delay(200);
  }

  if (i >= 30 && impulsCount == 1) {
    total_amount = total_amount + 1;
    impulsCount = 0;
    EEPROM.put(0, total_amount);
  }
  if (i >= 30 && impulsCount == 2) {
    total_amount = total_amount + 5;
    impulsCount = 0;
    EEPROM.put(0, total_amount);
  }
  if (i >= 30 && impulsCount == 3) {
    total_amount = total_amount + 10;
    impulsCount = 0;
    EEPROM.put(0, total_amount);
  }
  if (i >= 30 && impulsCount == 4) {
    impulsCount = 0;
  }
  if (i >= 30 && impulsCount == 5) {
    total_amount = total_amount + 20;
    impulsCount = 0;
    EEPROM.put(0, total_amount);
  }

  if (i >= 30 && impulsCount > 0) {
    if (isNeedsSelected) {
      needsTotal += total_amount;
      total_amount = 0;
      EEPROM.put(4, needsTotal);
      Serial.println("Money going to Needs compartment");
    } else if (isEmergencySelected) {
      emergencyTotal += total_amount;
      total_amount = 0;
      EEPROM.put(8, emergencyTotal);
      Serial.println("Money going to Emergency compartment");
    }
    impulsCount = 0;
    EEPROM.put(0, total_amount);
  }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Needs Total:");
  lcd.setCursor(0, 1);
  lcd.print(needsTotal, 2);

  delay(2000);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Emergency Total:");
  lcd.setCursor(0, 1);
  lcd.print(emergencyTotal, 2);

  delay(2000);
}

void parseData(String buff){
  Serial.println(buff);

  unsigned int len, index;
  
  index = buff.indexOf("\r");
  buff.remove(0, index+2);
  buff.trim();

  if(buff != "OK"){
    index = buff.indexOf(":");
    String cmd = buff.substring(0, index);
    cmd.trim();
    
    buff.remove(0, index+2);
    
    if(cmd == "+CMTI"){
      index = buff.indexOf(",");
      String temp = buff.substring(index+1, buff.length()); 
      temp = "AT+CMGR=" + temp + "\r"; 
      sim800.println(temp);
    }
    else if(cmd == "+CMGR"){
      extractSms(buff);
      
      if(senderNumber == PHONE){
        doAction();
        sim800.println("AT+CMGD=1,4");
        delay(1000);
        sim800.println("AT+CMGDA= \"DEL ALL\"");
        delay(1000);
      }
    }
  }
}

void extractSms(String buff){
  unsigned int index;
  
  index = buff.indexOf(",");
  smsStatus = buff.substring(1, index-1); 
  buff.remove(0, index+2);
  
  senderNumber = buff.substring(0, 13);
  buff.remove(0, 19);
  
  receivedDate = buff.substring(0, 20);
  buff.remove(0, buff.indexOf("\r"));
  buff.trim();
  
  index =buff.indexOf("\n\r");
  buff = buff.substring(0, index);
  buff.trim();
  msg = buff;
  buff = "";
  msg.toLowerCase();
}

void doAction(){
  if(msg == "lock"){  
    digitalWrite(RELAY_1, LOW);
    Reply("Piggy bank door is locked.");
  }
  else if(msg == "unlock"){
    digitalWrite(RELAY_1, HIGH);
    Reply("Piggy bank door is unlocked.");
  }

  smsStatus = "";
  senderNumber="";
  receivedDate="";
  msg="";  
}

void Reply(String text){
  sim800.print("AT+CMGF=1\r");
  delay(1000);
  sim800.print("AT+CMGS=\""+PHONE+"\"\r");
  delay(1000);
  sim800.print(text);
  delay(100);
  sim800.write(0x1A);
  delay(1000);
}
