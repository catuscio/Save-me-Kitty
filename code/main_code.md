# 메인 코드
```c
/*---------- macro ----------*/
#define APP_KEY "1fafcdf5ce97c4ccd2ec0faffbe8b20d"
#define REFRESH_TOKEN "G-Vk3R74XUeF2qmB6WqtItulAcgbjppwKQvGYa6ZCiolkAAAAYikMAMZ"
#define dataIn 18
#define cs 19
#define clk 21
#define gas A0

/*---------- Libraries ----------*/
#include <WiFi.h>
#include <HTTPClient.h>
#include <base64.h>
#include "LedControl.h"
#include "BluetoothSerial.h"

/*---------- WIFI ID & PW ----------*/
const char *ssid = "testpilot";
const char *password = "smarcle2017";

// kakao api token
const char *host = "https://kapi.kakao.com/v2/api/talk/memo/default/send";
String access_token;

/*---------- pin allocation ----------*/
int green = 5;
int red = 4;
int buz = 27;
int pir1 = 25;
int pir2 = 26;
int switchon = 32;

/*---------- global ----------*/
unsigned long past = 0;
float gasValue;

/*---------- Dot Matrix control ----------*/
LedControl lc = LedControl(dataIn, clk, cs, 2);

// emotions
byte smile[8] =
{  
    B00000000,
    B00000000,
    B00011000,
    B00100100,
    B01000010,
    B00000000,
    B00000000,
    B00000000  
};

byte angry[8] =
{ 
    B00000000,
    B00000000,
    B00000000,
    B01111110,
    B00000000,
    B00000000,
    B00000000,
    B00000000
};

/*cuteface*/
byte right_cute1[8] =
{
    B00000000,
    B00010000,
    B00100000,
    B01000000,
    B01000000,
    B00100000,
    B00010000,
    B00000000  
};

byte left_cute1[8] = 
{
    B00000000,
    B00100000,
    B00010000,
    B00001000,
    B00001000,
    B00010000,
    B00100000,
    B00000000
};

byte right_cute2[8] =
{
    B00000000,
    B00000100,
    B00001000,
    B00010000,
    B00010000,
    B00001000,
    B00000100,
    B00000000  
};

byte left_cute2[8] = 
{
    B00000000,
    B00001000,
    B00000100,
    B00000010,
    B00000010,
    B00000100,
    B00001000,
    B00000000
};

byte sad[8] = 
{
    B00000000,
    B00000000,
    B01111110,
    B00100100,
    B00100100,
    B00100100,
    B00000000,
    B00000000 
};

byte heart[8] =
  {
    B00000000,
    B00100100,
    B01111110,
    B01111110,
    B00111100,
    B00011000,
    B00000000,
    B00000000
  };

byte close_eyes[8] =
  {  
    B00000000,
    B00000000,
    B01000010,
    B00100100,
    B00011000,
    B00000000,
    B00000000,
    B00000000,
  };

byte open_eyes[8] =
  {
    B00000000,
    B00011000,
    B00100100,
    B00100100,
    B00100100,
    B00011000,
    B00000000,
    B00000000,
 };

///////////////////////////////////////////

/*---------- setup func ----------*/
void setup() {
  // Serial settings
  Serial.begin(115200);
  Serial.println(F("Hello, ESP32!\n"));

  // Pin Mode settings
  pinMode(green, OUTPUT);
  pinMode(red, OUTPUT);
  pinMode(switchon, INPUT);
  pinMode(buz, OUTPUT);
  pinMode(pir1, INPUT);
  pinMode(pir2, INPUT);

  // LED settings
  lc.shutdown(0, false);
  lc.shutdown(1, false);
  lc.setIntensity(0, 7);
  lc.setIntensity(1, 7);
  lc.clearDisplay(0);
  lc.clearDisplay(1);

  // WIFI settings
  WiFi.begin(ssid, password);
  Serial.println("Connecting");
  while (WiFi.status() != WL_CONNECTED)
  {
    Serial.print(".");
    delay(250);
  }
  Serial.println("");
  Serial.print("Connected to WiFi network with IP Address: ");
  Serial.println(WiFi.localIP());
}

/*---------- main func ----------*/
void loop() {
  int i = 0;
  int j;
  int cnt = 0;
  int p1 = digitalRead(pir1);
  int p2 = digitalRead(pir2);
  int pb = digitalRead(switchon);
  unsigned long emer;

  gasValue = analogRead(gas);
  Serial.println(gasValue);

  /*---------- touch sensing ----------*/
  int touchValue = touchRead(33);

  if (touchValue < 70){
    digitalWrite(red, LOW);
    digitalWrite(buz, LOW);

    while(i < 3){    // cute eyes
      for (j = 0; j <8; j++){
        lc.setRow(0, j, right_cute1[j]);
        lc.setRow(1, j, left_cute1[j]);
      }
      digitalWrite(green, HIGH);
      delay(300);   

      for (j = 0; j <8; j++) {
        lc.setRow(0, j, right_cute2[j]);
        lc.setRow(1, j, left_cute2[j]);
      }
      digitalWrite(green, LOW);
      delay(300);
      i++;
    }

    // heart emotion
    for (j = 0; j <8; j++){
      lc.setRow(0, j, heart[j]);
      lc.setRow(1, j, heart[j]);
    }
    digitalWrite(green, HIGH);

    // send kakao message
    if ( update_access_token() == true ) {
      send_message3();
    }
  }

  /*---------- gas sensing ----------*/
  if (gasValue > 300){
    while(1){
      Serial.println(F("*****Fire!!!!!*****"));
      digitalWrite(green, LOW);
      digitalWrite(red, HIGH);
      digitalWrite(buz, HIGH);

      // angry face
      for (j = 0; j <8; j++) {
        lc.setRow(0, j, angry[j]);
        lc.setRow(1, j, angry[j]);
      }

      // send kakao message
      if ( update_access_token() == true ) {
          if (cnt == 0) send_message1();
          cnt++;    // only once
      }
    }
  }

  /*---------- human motion not detected ----------*/
  if (p1 == LOW && p2 == LOW) {
    digitalWrite(green, LOW);
    digitalWrite(red, HIGH);
    digitalWrite(buz, HIGH);

    emer = millis();

    // surprised face
    for (j = 0; j <8; j++)   {
      lc.setRow(0, j, open_eyes[j]);
      lc.setRow(1, j, open_eyes[j]);
    }

    if (emer - past >= 5000){
      // sad face
      for (j = 0; j <8; j++)   {
        lc.setRow(0, j, sad[j]);
        lc.setRow(1, j, sad[j]);
      }

      // send kakao message
      if ( update_access_token() == true ) {
        if (cnt == 0) send_message2();
        cnt++;
      }
    }
  }
  /*---------- human motion detected ----------*/
  else {
    digitalWrite(green, HIGH);
    digitalWrite(red, LOW);
    digitalWrite(buz, LOW);
    past = millis();
    cnt = 0;

    // default face
    for (j = 0; j <8; j++) {
      lc.setRow(0, j, smile[j]);
      lc.setRow(1, j, smile[j]);
    }
  }

  // kakao API
  if ( Serial.available() )
  {
    int id = Serial.read();

    if ( id == '1' )
    {
      send_message1();
      send_message2();
    }
    // token 갱신하기 6시간 마다 갱신해야 함
    if ( id == '2' )
    {
      if ( update_access_token() == true )
      {
        send_message1();
        send_message2();
      }
    }
  }
}

/*---------- Kakao API funcs ----------*/
String urlencode(String str)  //
{
  String encodedString = "";
  char c;
  char code0;
  char code1;
  char code2;
  for (int i = 0; i < str.length(); i++) {
    c = str.charAt(i);
    if (c == ' ') {
      encodedString += '+';
    } else if (isalnum(c)) {
      encodedString += c;
    } else {
      code1 = (c & 0xf) + '0';
      if ((c & 0xf) > 9) {
        code1 = (c & 0xf) - 10 + 'A';
      }
      c = (c >> 4) & 0xf;
      code0 = c + '0';
      if (c > 9) {
        code0 = c - 10 + 'A';
      }
      code2 = '\0';
      encodedString += '%';
      encodedString += code0;
      encodedString += code1;
      //encodedString+=code2;
    }
    yield();
  }
  return encodedString;
}

/*---------- kakao messages ----------*/
// fire alert
void send_message1()
{
  HTTPClient http;
  if (!http.begin(host))
  {
    Serial.println("\nfailed to begin http\n");
  }
  http.addHeader("Authorization", "Bearer " + access_token); // <== 카카오 OPEN API의 Access Token을 입력하세요
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");

  int http_code;


  String data = "template_object={\"object_type\": \"text\",\"text\": \"거주지 내 화재 발생!!!\",\"link\": {\"web_url\": \"https://www.naver.com\",\"mobile_web_url\": \"https://www.naver.com\"},\"button_title\": \"바로 확인\"}";
  Serial.println(data);
  http_code = http.sendRequest("POST", data);
  Serial.print("HTTP Response code: ");
  Serial.println(http_code);

  String response;
  if (http_code > 0)
  {

    response = http.getString();
    Serial.println(response);
  }
  else
  {
  }
  http.end();
}

// emergency alert
void send_message2()
{
  HTTPClient http;
  if (!http.begin(host))
  {
    Serial.println("\nfailed to begin http\n");
  }
  http.addHeader("Authorization", "Bearer " + access_token); // <== 카카오 OPEN API의 Access Token을 입력하세요
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");

  int http_code;


  String data = "template_object={\"object_type\": \"text\",\"text\": \"거주자 응급상황 발생!!!\",\"link\": {\"web_url\": \"https://www.naver.com\",\"mobile_web_url\": \"https://www.naver.com\"},\"button_title\": \"바로 확인\"}";
  Serial.println(data);
  http_code = http.sendRequest("POST", data);
  Serial.print("HTTP Response code: ");
  Serial.println(http_code);

  String response;
  if (http_code > 0)
  {

    response = http.getString();
    Serial.println(response);
  }
  else
  {
  }
  http.end();
}

// pat pat
void send_message3()
{
  HTTPClient http;
  if (!http.begin(host))
  {
    Serial.println("\nfailed to begin http\n");
  }
  http.addHeader("Authorization", "Bearer " + access_token); // <== 카카오 OPEN API의 Access Token을 입력하세요
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");

  int http_code;


  String data = "template_object={\"object_type\": \"text\",\"text\": \"고먐미가 기뻐하고 있어요///ㅇㅅㅇ///\",\"link\": {\"web_url\": \"https://www.naver.com\",\"mobile_web_url\": \"https://www.naver.com\"},\"button_title\": \"바로 확인\"}";
  Serial.println(data);
  http_code = http.sendRequest("POST", data);
  Serial.print("HTTP Response code: ");
  Serial.println(http_code);

  String response;
  if (http_code > 0)
  {

    response = http.getString();
    Serial.println(response);
  }
  else
  {
  }
  http.end();
}

String extract_string(String str, String start_string, String end_string)
{
  int index1 = str.indexOf(start_string) + start_string.length();
  int index2 = str.indexOf(end_string, index1);
  String value = str.substring(index1, index2);
  return value;
}

bool update_access_token()
{
  HTTPClient http;
  String url = "https://kauth.kakao.com/oauth/token";
  if (!http.begin(url))
  {
    Serial.println("\nfailed to begin http\n");
  }
  int http_code;

  String client_id = String(APP_KEY);
  String refresh_token = String(REFRESH_TOKEN);
  String data = "grant_type=refresh_token&client_id=" + client_id + "&refresh_token=" + refresh_token; // <== 리프레쉬 토큰을 입력하세요.
  Serial.println(data);
  http_code = http.POST(data);
  Serial.print("HTTP Response code: ");
  Serial.println(http_code);

  String response;
  if (http_code > 0)
  {

    response = http.getString();
    Serial.println(response);
    access_token = extract_string(response, "{\"access_token\":\"", "\"");
    http.end();
    return true;
  }
  else
  {
  }
  http.end();
  return false;
}
```
- 카카오 API 사용법 및 코드 출처(https://blog.naver.com/mapes_khkim/222312839614)
