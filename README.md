TRANSMITTER - Glove

#include <esp_now.h>
#include <WiFi.h>

uint8_t receiverMAC[] = {0x28, 0x05, 0xA5, 0x32, 0x42, 0x5C};

//  Flex Sensor Pin Definitions (ESP32 ADC pins)
#define FLEX_PIN_1  34  // VP  - Finger 1 (Pinky)
#define FLEX_PIN_2  35   // VN  - Finger 2 (Ring)
#define FLEX_PIN_3  32   //     - Finger 3 (Middle)
#define FLEX_PIN_4  33   //     - Finger 4 (Index)
#define FLEX_PIN_5  27   //     - Finger 5 (Thumb)

//  Calibration Values
// Adjust these MIN/MAX values by reading your flex 
#define FLEX1_MIN 1370   
#define FLEX1_MAX 1290
#define FLEX2_MIN 1400   
#define FLEX2_MAX 1300
#define FLEX3_MIN 1210   
#define FLEX3_MAX 1150
#define FLEX4_MIN 1390   
#define FLEX4_MAX 1250
#define FLEX5_MIN 1400   
#define FLEX5_MAX 1350

// Servo angle range
#define SERVO_OPEN   0    // degrees 
#define SERVO_CLOSED 180   // degrees 

// Data Structure (must match Receiver exactly)
typedef struct {
  int flex1;
  int flex2;
  int flex3;
  int flex4;
  int flex5;
} GloveData;

GloveData gloveData;
esp_now_peer_info_t peerInfo;

// Callback 
void OnDataSent(const uint8_t *mac_addr, esp_now_send_status_t status) {
  // Uncomment below line for send debug (may slow loop slightly)
  // Serial.println(status == ESP_NOW_SEND_SUCCESS ? "Sent OK" : "Send FAIL");
}

// Read and map a flex sensor to servo angle

int readFlex(int pin, int rawMin, int rawMax) {
  int raw = analogRead(pin);
  int angle = map(raw, rawMin, rawMax, SERVO_OPEN, SERVO_CLOSED);
  return constrain(angle, SERVO_OPEN, SERVO_CLOSED);
}

void setup() {
  Serial.begin(115200);

  // Init WiFi
  WiFi.mode(WIFI_STA);
  WiFi.disconnect();

  Serial.print("TRANSMITTER MAC: ");
  Serial.println(WiFi.macAddress());

  // Init ESP-NOW
  if (esp_now_init() != ESP_OK) {
    Serial.println("ESP-NOW init failed!");
    return;
  }

  esp_now_register_send_cb((esp_now_send_cb_t)OnDataSent);

  // Register receiver as peer
  memcpy(peerInfo.peer_addr, receiverMAC, 6);
  peerInfo.channel = 0;
  peerInfo.encrypt = false;

  if (esp_now_add_peer(&peerInfo) != ESP_OK) {
    Serial.println("Failed to add peer!");
    return;
  }

  Serial.println("Transmitter ready. Sending glove data...");
}

void loop() {
  // Read all 5 flex sensors and map to servo angles
  gloveData.flex1 = readFlex(FLEX_PIN_1, FLEX1_MIN, FLEX1_MAX);
  gloveData.flex2 = readFlex(FLEX_PIN_2, FLEX2_MIN, FLEX2_MAX);
  gloveData.flex3 = readFlex(FLEX_PIN_3, FLEX3_MIN, FLEX3_MAX);
  gloveData.flex4 = readFlex(FLEX_PIN_4, FLEX4_MIN, FLEX4_MAX);
  gloveData.flex5 = readFlex(FLEX_PIN_5, FLEX5_MIN, FLEX5_MAX);

  // Send to receiver
  esp_now_send(receiverMAC, (uint8_t *)&gloveData, sizeof(gloveData));

  // Debug output
  Serial.printf("F1:%d  F2:%d  F3:%d  F4:%d  F5:%d\n",
    gloveData.flex1, gloveData.flex2, gloveData.flex3,
    gloveData.flex4, gloveData.flex5);

  delay(20); // ~50Hz update rate — smooth and responsive
}
