# GPS-tracking-system-using-A9G-module
A GPS tracking system uses the Global Positioning System (GPS) to determine and monitor the precise location of a vehicle, person, or asset. These systems collect location data from GPS satellites and often transmit it to a central database or device where it can be monitored in real time. 
# code 
#include "WiFi.h"
#define SOS 4
#define SLEEP_PIN 2 // Make this pin HIGH to make A9G board go to sleep mode

String SOS_NUM = "XXXX"; // Add a number on which you want to receive a call or SMS
int SOS_Time = 5; // Press the button for 5 sec

// Necessary Variables
boolean stringComplete = false;
String inputString = "";
String fromGSM = "";
bool CALL_END = 1;
String response = "";
String res = "";
int c = 0;

void setup() {
  Serial.begin(115200); // For Serial Monitor
  Serial1.begin(115200, SERIAL_8N1, 3, 1); // For A9G Board

  // Making Radio OFF for power saving
  WiFi.mode(WIFI_OFF); // WiFi OFF
  btStop(); // Bluetooth OFF

  pinMode(SOS, INPUT_PULLUP);
  pinMode(SLEEP_PIN, OUTPUT);

  // Waiting for A9G to setup everything for 20 sec
  delay(2000);
  digitalWrite(SLEEP_PIN, LOW); // Sleep Mode OFF

  Serial1.println("AT"); // Just Checking
  delay(1000);
  Serial1.println("AT+GPS=1"); // Turning ON GPS
  delay(1000);
  Serial1.println("AT+GPSLP=2"); // GPS low power
  delay(1000);
  Serial1.println("AT+SLEEP=1"); // Configuring Sleep Mode to 1
  delay(1000);
  Serial1.println("AT+CMGF=1");
  delay(1000);
  Serial1.println("AT+CSMP=17,167,0,0");
  delay(1000);
  Serial1.println("AT+CPMS=\"SM\",\"ME\",\"SM\"");
  delay(1000);
  Serial1.println("AT+SNFS=2");
  delay(1000);
  Serial1.println("AT+CLVL=8");
  delay(1000);

  digitalWrite(SLEEP_PIN, HIGH); // Sleep Mode ON
}

void loop() {
  // Listen from GSM Module
  if (Serial1.available()) {
    char inChar = Serial1.read();
    if (inChar == '\n') {
      // Check the state
      if (fromGSM.startsWith("SEND LOCATION")) {
        Get_gmap_link(false); // Send Location without Call
        digitalWrite(SLEEP_PIN, HIGH); // Sleep Mode ON
      } else if (fromGSM == "RING\r") {
        digitalWrite(SLEEP_PIN, LOW); // Sleep Mode OFF
        Serial.println("---------ITS RINGING-------");
        Serial1.println("ATA"); // Automatically answer the call
        Get_gmap_link(false); // Send Location without Call
      } else if (fromGSM == "NO CARRIER\r") {
        Serial.println("---------CALL ENDS-------");
        CALL_END = 1;
        digitalWrite(SLEEP_PIN, HIGH); // Sleep Mode ON
      } else if (fromGSM.startsWith("+CMT:")) {
        // Handle incoming SMS
        String phoneNumber = parsePhoneNumber(fromGSM);
        fromGSM = ""; // Clear the buffer to read the SMS content

        // Read the SMS content
        while (Serial1.available()) {
          char inChar = Serial1.read();
          if (inChar == '\n') {
            if (fromGSM.indexOf("SEND LOCATION") != -1) {
              SOS_NUM = phoneNumber;
              Get_gmap_link(false); // Send Location without Call
              break;
            }
            fromGSM = "";
          } else {
            fromGSM += inChar;
          }
          delay(20);
        }
      }
      // Write the actual response
      Serial.println(fromGSM);
      // Clear the buffer
      fromGSM = "";
    } else {
      fromGSM += inChar;
    }
    delay(20);
  }

  // Read from port 0, send to port 1:
  if (Serial.available()) {
    int inByte = Serial.read();
    Serial1.write(inByte);
  }

  // When SOS button is pressed and held for 5 seconds
  if (digitalRead(SOS) == LOW && CALL_END == 1) {
    Serial.print("SOS Button Pressed.."); // Waiting for 5 sec
    for (c = 0; c < SOS_Time; c++) {
      Serial.println((SOS_Time - c));
      delay(1000);
      if (digitalRead(SOS) == HIGH)
        break;
    }
    if (c == 5) {
      Get_gmap_link(true); // Send Location with Call
    }
  }

  // Only write a full message to the GSM module
  if (stringComplete) {
    Serial1.print(inputString);
    inputString = "";
    stringComplete = false;
  }
}

// Getting Location and making Google Maps link of it. Also making a call if needed
void Get_gmap_link(bool makeCall) {
  digitalWrite(SLEEP_PIN, LOW);
  delay(1000);
  Serial1.println("AT+LOCATION=2");

  while (!Serial1.available());
  while (Serial1.available()) {
    char add = Serial1.read();
    res = res + add;
    delay(1);
  }

  Serial.print("Received Data: "); Serial.println(res); // Print the received data for debugging

  // Extract the location data
  int startIndex = res.indexOf("Lat:");
  int endIndex = res.indexOf("\r\n", startIndex);
  if (startIndex != -1 && endIndex != -1) {
    response = res.substring(startIndex + 4, endIndex);
  }

  Serial.print("Parsed Location Data: "); Serial.println(response); // Print the parsed location data

  if (response.indexOf("GPS NOT") != -1) {
    Serial.println("No Location data");
    // Sending SMS without any location
    Serial1.println("AT+CMGF=1");
    delay(1000);
    Serial1.println("AT+CMGS=\"" + SOS_NUM + "\"\r");
    delay(1000);
    Serial1.println("Unable to fetch location. Please try again");
    delay(1000);
    Serial1.write((char)26);
    delay(1000);
  } else {
    int i = response.indexOf(',');
    String lat = response.substring(0, i);
    String longi = response.substring(i + 1);

    Serial.println("Latitude: " + lat);
    Serial.println("Longitude: " + longi);

    String Gmaps_link = "http://maps.google.com/maps?q=" + lat + "," + longi;

    // Sending SMS with Google Maps Link with our Location
    Serial1.println("AT+CMGF=1");
    delay(1000);
    Serial1.println("AT+CMGS=\"" + SOS_NUM + "\"\r");
    delay(1000);
    Serial1.print("I'm here " + Gmaps_link);
    delay(1000);
    Serial1.write((char)26); // CTRL+Z to send the SMS
    delay(1000);
    Serial1.println("AT+CMGD=1,4"); // Delete stored SMS to save memory
    delay(5000);
  }

  response = "";
  res = "";

  if (makeCall) {
    Serial.println("Calling Now");
    Serial1.println("ATD" + SOS_NUM + ";"); // Ensure the correct command format for calling
    CALL_END = 0;
  }
}

// Function to parse phone number from incoming SMS
String parsePhoneNumber(String gsmData) {
  int start = gsmData.indexOf("\"") + 1;
  int end = gsmData.indexOf("\"", start);
  return gsmData.substring(start, end);
}
