#include "esp_camera.h"
#include <WiFi.h>
#include "esp_timer.h"
#include "img_converters.h"
#include "Arduino.h"
#include "fb_gfx.h"
#include "soc/soc.h"           // Disable brownout problems
#include "soc/rtc_cntl_reg.h"  // Disable brownout problems
#include "esp_http_server.h"
#include <ESP32Servo.h>

// ================== Wi-Fi Credentials ==================
const char* ssid     = "***************";
const char* password = "**************";

// ================== Camera Model ==================
#define PART_BOUNDARY "123456789000000000000987654321"
#define CAMERA_MODEL_AI_THINKER

#if defined(CAMERA_MODEL_AI_THINKER)
  #define PWDN_GPIO_NUM     32
  #define RESET_GPIO_NUM    -1
  #define XCLK_GPIO_NUM     0
  #define SIOD_GPIO_NUM     26
  #define SIOC_GPIO_NUM     27
  #define Y9_GPIO_NUM       35
  #define Y8_GPIO_NUM       34
  #define Y7_GPIO_NUM       39
  #define Y6_GPIO_NUM       36
  #define Y5_GPIO_NUM       21
  #define Y4_GPIO_NUM       19
  #define Y3_GPIO_NUM       18
  #define Y2_GPIO_NUM       5
  #define VSYNC_GPIO_NUM    25
  #define HREF_GPIO_NUM     23
  #define PCLK_GPIO_NUM     22
#else
  #error "Camera model not selected"
#endif

// ================== Servo Config ==================
#define SERVO_1   14
#define SERVO_STEP 10
Servo servo;
int servoPos = 90;

// ================== Streaming constants ==================
static const char* _STREAM_CONTENT_TYPE = "multipart/x-mixed-replace;boundary=" PART_BOUNDARY;
static const char* _STREAM_BOUNDARY     = "\r\n--" PART_BOUNDARY "\r\n";
static const char* _STREAM_PART         = "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n";

httpd_handle_t camera_httpd = NULL;
httpd_handle_t stream_httpd = NULL;

// ================== Web Page ==================
static const char PROGMEM INDEX_HTML[] = R"rawliteral(
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Borewell Rescue System</title>
<style>
  body { font-family: Arial, sans-serif; margin:0; padding:0; background:#000; color:#fff; text-align:center; }
  .header { background:#ff0000; color:#fff; padding:20px 0; }
  .header h1 { margin:0; }
  .container { display:flex; flex-direction:column; align-items:center; justify-content:flex-start; min-height:100vh; padding-top:40px; }
  .stream { margin:10px auto; max-width:100%; border:2px solid #fff000; border-radius:10px; }
  .controls { margin-top:20px; }
  .button { background:#fff000; border:none; color:#000; padding:15px 30px; margin:5px; font-size:16px; cursor:pointer; border-radius:5px; transition:background-color 0.3s; }
  .button:hover { background:#cccc00; }
  .dropdown { margin-top:15px; padding:10px; font-size:16px; background:#fff000; color:#000; border:1px solid #000; border-radius:5px; }
  @media screen and (max-width: 600px) {
    .container { padding:10px; }
    .button { padding:10px 20px; font-size:14px; }
  }
  body.dark-mode { background:#333; color:#fff; }
  .dark-mode .header { background:#ff6600; }
  #log { margin-top:20px; text-align:left; max-height:200px; overflow-y:auto; width:90%; }
</style>
</head>
<body>
  <div class="header">
    <h1>Borewell Rescue System</h1>
  </div>

  <div class="container">
    <!-- Camera Stream -->
    <img src="" id="photo" class="stream" alt="Camera Stream">

    <!-- Camera / Flash Controls -->
    <div class="controls">
      <button class="button" onclick="sendCommand('left')">Left</button>
      <button class="button" onclick="sendCommand('right')">Right</button>
      <button class="button" onclick="sendCommand('flash_on')">Flash On</button>
      <button class="button" onclick="sendCommand('flash_off')">Flash Off</button>
    </div>

    <!-- Screen Size Control -->
    <select id="screenSize" class="dropdown" onchange="changeScreenSize()">
      <option value="100">Default</option>
      <option value="75">75%</option>
      <option value="50">50%</option>
      <option value="25">25%</option>
    </select>

    <!-- Status Indicator -->
    <div id="status">Camera Status: <span id="cameraStatus">Loading...</span></div>

    <!-- Dark Mode Toggle -->
    <button class="button" onclick="toggleMode()">Toggle Dark Mode</button>

    <!-- Real-Time Log -->
    <div id="log"></div>
  </div>

<script>
  // Update the camera stream source
  window.onload = () => {
    const photo = document.getElementById("photo");
    const cameraStatus = document.getElementById("cameraStatus");
    const log = document.getElementById("log");

    setTimeout(() => {
      // Stream server runs on port 81
      const base = window.location.origin.replace(/:\\d+$/, '');
      photo.src = `${base}:81/stream`;
      cameraStatus.innerText = "Online";
      log.innerHTML += "<p>[INFO] Camera Stream started.</p>";
    }, 1000);

    photo.onerror = () => {
      cameraStatus.innerText = "Offline";
      log.innerHTML += "<p>[ERROR] Camera Stream failed.</p>";
    };
  };

  // Send command to ESP32
  function sendCommand(cmd) {
    fetch(`/action?go=${cmd}`).catch(()=>{});
    const log = document.getElementById("log");
    log.innerHTML += `<p>[COMMAND] ${cmd}</p>`;
  }

  // Change screen size of the camera stream
  function changeScreenSize() {
    const photo = document.getElementById("photo");
    const screenSize = document.getElementById("screenSize").value;
    photo.style.width = screenSize + "%";
    const log = document.getElementById("log");
    log.innerHTML += `<p>[INFO] Screen size changed to ${screenSize}%</p>`;
  }

  // Toggle dark mode
  function toggleMode() {
    document.body.classList.toggle('dark-mode');
    const log = document.getElementById("log");
    log.innerHTML += "<p>[INFO] Dark mode toggled.</p>";
  }
</script>
</body>
</html>
)rawliteral";

// ================== HTTP Handlers ==================
static esp_err_t index_handler(httpd_req_t *req){
  httpd_resp_set_type(req, "text/html");
  return httpd_resp_send(req, (const char *)INDEX_HTML, strlen(INDEX_HTML));
}

static esp_err_t stream_handler(httpd_req_t *req){
  camera_fb_t * fb = NULL;
  esp_err_t res = ESP_OK;
  size_t _jpg_buf_len = 0;
  uint8_t * _jpg_buf = NULL;
  char part_buf[64];

  res = httpd_resp_set_type(req, _STREAM_CONTENT_TYPE);
  if(res != ESP_OK){ return res; }

  while(true){
    fb = esp_camera_fb_get();
    if (!fb) {
      Serial.println("Camera capture failed");
      res = ESP_FAIL;
    } else {
      if(fb->format != PIXFORMAT_JPEG){
        bool jpeg_converted = frame2jpg(fb, 80, &_jpg_buf, &_jpg_buf_len);
        esp_camera_fb_return(fb);
        fb = NULL;
        if(!jpeg_converted){
          Serial.println("JPEG compression failed");
          res = ESP_FAIL;
        }
      } else {
        _jpg_buf_len = fb->len;
        _jpg_buf = fb->buf;
      }
    }

    if(res == ESP_OK){
      size_t hlen = snprintf(part_buf, sizeof(part_buf), _STREAM_PART, _jpg_buf_len);
      res = httpd_resp_send_chunk(req, (const char *)part_buf, hlen);
    }
    if(res == ESP_OK){
      res = httpd_resp_send_chunk(req, (const char *)_jpg_buf, _jpg_buf_len);
    }
    if(res == ESP_OK){
      res = httpd_resp_send_chunk(req, _STREAM_BOUNDARY, strlen(_STREAM_BOUNDARY));
    }

    if(fb){
      esp_camera_fb_return(fb);
      fb = NULL;
      _jpg_buf = NULL;
    } else if(_jpg_buf){
      free(_jpg_buf);
      _jpg_buf = NULL;
    }

    if(res != ESP_OK){
      break;
    }
  }
  return res;
}

static esp_err_t cmd_handler(httpd_req_t *req){
  char*  buf;
  size_t buf_len;
  char   variable[32] = {0};

  buf_len = httpd_req_get_url_query_len(req) + 1;
  if (buf_len > 1) {
    buf = (char*)malloc(buf_len);
    if(!buf){ httpd_resp_send_500(req); return ESP_FAIL; }

    if (httpd_req_get_url_query_str(req, buf, buf_len) == ESP_OK) {
      if (httpd_query_key_value(buf, "go", variable, sizeof(variable)) != ESP_OK) {
        free(buf); httpd_resp_send_404(req); return ESP_FAIL;
      }
    } else { free(buf); httpd_resp_send_404(req); return ESP_FAIL; }
    free(buf);
  } else { httpd_resp_send_404(req); return ESP_FAIL; }

  int res = 0;
  if(!strcmp(variable, "left")) {
    if(servoPos <= 170) { servoPos += SERVO_STEP; servo.write(servoPos); delay(10); }
    Serial.printf("Servo: %d (Left)\n", servoPos);
  }
  else if(!strcmp(variable, "right")) {
    if(servoPos >= 10) { servoPos -= SERVO_STEP; servo.write(servoPos); delay(10); }
    Serial.printf("Servo: %d (Right)\n", servoPos);
  }
  else if (!strcmp(variable, "flash_on")) {
    digitalWrite(4, HIGH);
    Serial.println("Flashlight ON");
  }
  else if (!strcmp(variable, "flash_off")) {
    digitalWrite(4, LOW);
    Serial.println("Flashlight OFF");
  }
  else {
    res = -1;
  }

  if(res){ return httpd_resp_send_500(req); }
  httpd_resp_set_hdr(req, "Access-Control-Allow-Origin", "*");
  return httpd_resp_send(req, NULL, 0);
}

// ================== Server Boot ==================
void startCameraServer(){
  httpd_config_t config = HTTPD_DEFAULT_CONFIG();
  config.server_port = 80; // index + commands

  httpd_uri_t index_uri = { .uri = "/", .method = HTTP_GET, .handler = index_handler, .user_ctx = NULL };
  httpd_uri_t cmd_uri   = { .uri = "/action", .method = HTTP_GET, .handler = cmd_handler, .user_ctx = NULL };
  httpd_uri_t stream_uri= { .uri = "/stream", .method = HTTP_GET, .handler = stream_handler, .user_ctx = NULL };

  if (httpd_start(&camera_httpd, &config) == ESP_OK) {
    httpd_register_uri_handler(camera_httpd, &index_uri);
    httpd_register_uri_handler(camera_httpd, &cmd_uri);
  }

  // Stream server on port 81
  config.server_port += 1;
  config.ctrl_port   += 1;
  if (httpd_start(&stream_httpd, &config) == ESP_OK) {
    httpd_register_uri_handler(stream_httpd, &stream_uri);
  }
}

// ================== Setup & Loop ==================
void setup() {
  WRITE_PERI_REG(RTC_CNTL_BROWN_OUT_REG, 0); // disable brownout detector

  // Servo
  servo.attach(SERVO_1, 1000, 2000);
  servo.write(servoPos);

  // Flash LED (GPIO4)
  pinMode(4, OUTPUT);
  digitalWrite(4, LOW);

  Serial.begin(115200);
  Serial.setDebugOutput(false);

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;
  config.pin_d0       = Y2_GPIO_NUM;
  config.pin_d1       = Y3_GPIO_NUM;
  config.pin_d2       = Y4_GPIO_NUM;
  config.pin_d3       = Y5_GPIO_NUM;
  config.pin_d4       = Y6_GPIO_NUM;
  config.pin_d5       = Y7_GPIO_NUM;
  config.pin_d6       = Y8_GPIO_NUM;
  config.pin_d7       = Y9_GPIO_NUM;
  config.pin_xclk     = XCLK_GPIO_NUM;
  config.pin_pclk     = PCLK_GPIO_NUM;
  config.pin_vsync    = VSYNC_GPIO_NUM;
  config.pin_href     = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn     = PWDN_GPIO_NUM;
  config.pin_reset    = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;

  if(psramFound()){
    config.frame_size   = FRAMESIZE_VGA;
    config.jpeg_quality = 10;
    config.fb_count     = 2;
  } else {
    config.frame_size   = FRAMESIZE_SVGA;
    config.jpeg_quality = 12;
    config.fb_count     = 1;
  }

  // Camera init
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x\n", err);
    return;
  }

  // Orientation (adjust as needed)
  sensor_t *s = esp_camera_sensor_get();
  if(s != NULL){
    s->set_hmirror(s, 0);
    s->set_vflip(s, 1);
  }

  // Wi-Fi connection
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println();
  Serial.println("WiFi connected");
  Serial.print("Camera Stream Ready!\nGo to: http://");
  Serial.println(WiFi.localIP());

  // Start servers
  startCameraServer();
}

void loop() {
  // Nothing here — everything runs via HTTP handlers
}
