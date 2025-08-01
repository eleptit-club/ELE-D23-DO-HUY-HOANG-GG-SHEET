# ELE-D23-DO-HUY-HOANG-GG-SHEET

#include <SPI.h>
#include <MFRC522.h>
#define SS_PIN 16
#define RST_PIN 17
MFRC522 rfid(SS_PIN, RST_PIN);
String uidString;

#include "WiFi.h"
#include <HTTPClient.h>
const char* ssid = "Hoang.....";
const char* password = "khongcomatkhau";
String Web_App_URL = "https://script.google.com/macros/s/AKfycbxKIl9a55-KLRZk1jfh4pyDdnfuOMvwL6M3eG6gilONk_AfFGRcQT9nVR97QzMUSYitpw/exec";
#define On_Board_LED_PIN 2
#define MAX_STUDENTS 10

struct Student {
  String id;    // Cột A
  String code;  // Cột C: UID
  String name;  // Cột B
};
Student students[MAX_STUDENTS];
bool studentInOutState[MAX_STUDENTS]; // 0: vào, 1: ra
int studentCount = 0;
unsigned long timeDelay2 = millis();
const int ledIO = 4;
const int buzzer = 5;

void setup() {
  Serial.begin(115200);
  pinMode(buzzer, OUTPUT);
  digitalWrite(buzzer, HIGH);
  pinMode(ledIO, OUTPUT);
  pinMode(On_Board_LED_PIN, OUTPUT);

  Serial.println("-------------");
  Serial.println("WIFI mode : STA");
  WiFi.begin(ssid, password);
  while (WiFi.status() != WL_CONNECTED) {
    Serial.print(".");
    delay(1000);
  }
  Serial.println("\nĐã kết nối WiFi!");

  if (!readDataSheet()) Serial.println("Không đọc được dữ liệu từ Google Sheet!");

  for (int i = 0; i < studentCount; i++) {
    studentInOutState[i] = 0; // mặc định là "vào"
  }

  SPI.begin();
  rfid.PCD_Init();
}

void loop() {
  if (millis() - timeDelay2 > 500) {
    readUID();
    timeDelay2 = millis();
  }
}

void beep(int n, int d) {
  for (int i = 0; i < n; i++) {
    digitalWrite(buzzer, LOW);
    delay(d);
    digitalWrite(buzzer, HIGH);
    delay(d);
  }
}

void readUID() {
  if (!rfid.PICC_IsNewCardPresent() || !rfid.PICC_ReadCardSerial()) {
    return;
  }
  uidString = "";
  for (byte i = 0; i < rfid.uid.size; i++) {
    uidString += String(rfid.uid.uidByte[i] < 0x10 ? "0" : "") + String(rfid.uid.uidByte[i], HEX);
  }
  uidString.toUpperCase();
  Serial.println("Card UID: " + uidString);
  beep(1, 200);
  writeLogSheet();
}

bool readDataSheet() {
  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(On_Board_LED_PIN, HIGH);
    String Read_Data_URL = Web_App_URL + "?sts=read";
    Serial.println("-------------");
    Serial.println("Đọc dữ liệu từ Google Sheet...");
    Serial.println("URL: " + Read_Data_URL);

    HTTPClient http;
    http.begin(Read_Data_URL.c_str());
    http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS);
    int httpCode = http.GET();
    Serial.print("HTTP Status Code: ");
    Serial.println(httpCode);

    if (httpCode > 0) {
      String payload = http.getString();
      Serial.println("Payload: " + payload);
      
      studentCount = 0;
      int startIdx = 1;
      while (startIdx < payload.length() && studentCount < MAX_STUDENTS) {
        int endIdx = payload.indexOf(']', startIdx);
        if (endIdx == -1) break;

        String studentData = payload.substring(startIdx, endIdx + 1);
        studentData.replace("[", "");
        studentData.replace("]", "");
        studentData.replace("\"", "");

        int comma1 = studentData.indexOf(',');
        int comma2 = studentData.indexOf(',', comma1 + 1);
        if (comma1 == -1 || comma2 == -1) break;

        students[studentCount].id = studentData.substring(0, comma1);
        students[studentCount].name = studentData.substring(comma1 + 1, comma2);
        students[studentCount].code = studentData.substring(comma2 + 1);
        students[studentCount].code.trim();

        Serial.println("Đã lưu: ID=" + students[studentCount].id + ", Tên=" + students[studentCount].name + ", UID=" + students[studentCount].code);
        studentCount++;
        startIdx = payload.indexOf('[', endIdx) + 1;
        if (startIdx == 0) break;
      }
      Serial.println("Tổng số sinh viên: " + String(studentCount));
    }
    http.end();
    digitalWrite(On_Board_LED_PIN, LOW);
    return studentCount > 0;
  }
  return false;
}

void writeLogSheet() {
  int idx = getStudentIndexById(uidString);
  if (idx != -1) {
    String studentName = students[idx].name;
    bool currentState = studentInOutState[idx];

    Serial.println("Tên học sinh với UID " + uidString + ": " + studentName);
    String Send_Data_URL = Web_App_URL + "?sts=writelog&uid=" + uidString + "&name=" + urlencode(studentName);
    Send_Data_URL += "&inout=" + urlencode(currentState == 0 ? "VÀO" : "RA");

    Serial.println("-------------");
    Serial.println("Gửi log điểm danh lên Google Sheet...");
    Serial.println("URL: " + Send_Data_URL);

    HTTPClient http;
    http.begin(Send_Data_URL.c_str());
    http.setFollowRedirects(HTTPC_STRICT_FOLLOW_REDIRECTS);
    int httpCode = http.GET();
    Serial.print("HTTP Status Code: ");
    Serial.println(httpCode);
    if (httpCode > 0) {
      Serial.println("Payload: " + http.getString());
    }
    http.end();

    studentInOutState[idx] = !studentInOutState[idx]; // Đảo trạng thái
    //digitalWrite(ledIO, studentInOutState[idx]); // Đổi trạng thái LED (nếu cần)
  } else {
    Serial.println("Không tìm thấy học sinh với UID: " + uidString);
    beep(3, 500);
  }
}

int getStudentIndexById(String uid) {
  for (int i = 0; i < studentCount; i++) {
    String storedUID = students[i].code;
    storedUID.trim();
    if (storedUID == uid) {
      return i;
    }
  }
  return -1;
}

String urlencode(String str) {
  String encodedString = "";
  for (int i = 0; i < str.length(); i++) {
    char c = str.charAt(i);
    if (c == ' ') {
      encodedString += '+';
    } else if (isalnum(c)) {
      encodedString += c;
    } else {
      encodedString += '%' + String(c, HEX);
    }
  }
  return encodedString;
}
