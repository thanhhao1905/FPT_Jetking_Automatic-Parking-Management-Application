````C
#include <SPI.h>
#include <MFRC522.h>

// Pin Configuration
#define RST_PIN_ENTRY 21       
#define RST_PIN_EXIT  27       

#define SS_PIN_ENTRY  32        
#define SS_PIN_EXIT   33       

#define TO_8051_PIN_ENTRY 12  
#define SENIN_PIN 4          

#define TO_8051_PIN_EXIT 13   
#define SENOUT_PIN 2         


byte authorizedUIDs[][4] = {
  {165, 170, 232, 0},
  {150, 238, 19, 5}
};
#define NUM_AUTH_UIDS (sizeof(authorizedUIDs) / sizeof(authorizedUIDs[0]))


MFRC522 mfrc522Entry(SS_PIN_ENTRY, RST_PIN_ENTRY);
MFRC522 mfrc522Exit(SS_PIN_EXIT, RST_PIN_EXIT);

void setup() {
  Serial.begin(9600);


  pinMode(SENIN_PIN, OUTPUT);
  pinMode(TO_8051_PIN_ENTRY, OUTPUT);
  pinMode(SENOUT_PIN, OUTPUT);
  pinMode(TO_8051_PIN_EXIT, OUTPUT);


  digitalWrite(SENIN_PIN, LOW);
  digitalWrite(TO_8051_PIN_ENTRY, LOW);
  digitalWrite(SENOUT_PIN, LOW);
  digitalWrite(TO_8051_PIN_EXIT, LOW);


  SPI.begin();

 
  mfrc522Entry.PCD_Init();
  mfrc522Exit.PCD_Init();

  Serial.println("Dual RFID System Ready");
}

void loop() {
  checkRFID(mfrc522Entry, SS_PIN_ENTRY, SENIN_PIN, TO_8051_PIN_ENTRY, "Entry");
  checkRFID(mfrc522Exit, SS_PIN_EXIT, SENOUT_PIN, TO_8051_PIN_EXIT, "Exit");
  delay(50); // Giảm nhiễu đọc
}

void checkRFID(MFRC522 &reader, byte ssPin, byte controlPin, byte alarmPin, const char* readerName) {
  digitalWrite(ssPin, LOW);  

  if (!reader.PICC_IsNewCardPresent() || !reader.PICC_ReadCardSerial()) {
    digitalWrite(ssPin, HIGH); 
    return;
  }


  Serial.print(readerName);
  Serial.print(" UID: ");
  for (byte i = 0; i < reader.uid.size; i++) {
    Serial.print(reader.uid.uidByte[i], HEX);
    if (i < reader.uid.size - 1) Serial.print(":");
  }
  Serial.println();


  bool authorized = false;
  for (byte i = 0; i < NUM_AUTH_UIDS; i++) {
    bool match = true;
    for (byte j = 0; j < 4; j++) {
      if (reader.uid.uidByte[j] != authorizedUIDs[i][j]) {
        match = false;
        break;
      }
    }
    if (match) {
      authorized = true;
      break;
    }
  }


  if (authorized) {
    digitalWrite(controlPin, HIGH); 
    digitalWrite(alarmPin, LOW);     
    Serial.print(readerName);
    Serial.println(": Valid Card - Access Granted");
  } else {
    digitalWrite(alarmPin, HIGH);   
    digitalWrite(controlPin, LOW);   
    Serial.print(readerName);
    Serial.println(": Invalid Card - Alarm Triggered");
  }


  while (reader.PICC_IsNewCardPresent() || reader.PICC_ReadCardSerial()) {
    delay(100);
  }


  digitalWrite(controlPin, LOW);
  digitalWrite(alarmPin, LOW);


  reader.PICC_HaltA();
  reader.PCD_StopCrypto1();
  digitalWrite(ssPin, HIGH); 
}
