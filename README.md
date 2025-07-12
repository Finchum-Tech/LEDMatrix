basic template from chatgpt - needs much massaging for platformio integration:
// snake_esp32_configurable/src/main.cpp

#include <Arduino.h>
#include <Adafruit_NeoPixel.h>
#include <EEPROM.h>
#include <WiFiManager.h>
#include <WebServer.h>
#include <ArduinoOTA.h>
#include <FastLED.h> // for hsv2rgb_rainbow only
#include <Wire.h>
#include <WiiChuck.h>
#include <math.h>

// ==== Display Config ====
#define TILE_WIDTH     16
#define TILE_HEIGHT    16
#define TILES_X        6
#define TILES_Y        3
#define MATRIX_WIDTH   (TILE_WIDTH * TILES_X)
#define MATRIX_HEIGHT  (TILE_HEIGHT * TILES_Y)
#define NUM_LEDS       (MATRIX_WIDTH * MATRIX_HEIGHT)
#define LED_PIN        5

// ==== EEPROM ====
#define CONFIG_VERSION 1
#define EEPROM_SIZE 512

struct Config {
  uint8_t version = CONFIG_VERSION;
  float brightness = 0.2f;
  bool serpentineTiles = true;
  bool serpentineInTile = true;
  bool flipTiles = false;
  uint8_t difficulty = 1;
};

Config cfg;
Adafruit_NeoPixel strip(NUM_LEDS, LED_PIN, NEO_GRB + NEO_KHZ800);
WebServer server(80);
WiiChuck nunchuck;

// ==== Vector2 ====
struct Vector2 {
  float x, y;
};

// ==== Asteroids Entities ====
#define MAX_BULLETS 8
#define MAX_ASTEROIDS 12

struct Bullet {
  Vector2 pos;
  Vector2 vel;
  bool active;
};

struct Asteroid {
  Vector2 pos;
  Vector2 vel;
  int size;  // 2 = big, 1 = small
  bool active;
};

struct Ship {
  Vector2 pos;
  Vector2 vel;
  float angle;  // radians
};

Ship ship;
Bullet bullets[MAX_BULLETS];
Asteroid asteroids[MAX_ASTEROIDS];

// ==== EEPROM Load/Save ====
void loadConfig() {
  EEPROM.begin(EEPROM_SIZE);
  EEPROM.get(0, cfg);
  if (cfg.version != CONFIG_VERSION) {
    Serial.println("Config mismatch, resetting...");
    cfg = Config();
    EEPROM.put(0, cfg);
    EEPROM.commit();
  }
}

void saveConfig() {
  EEPROM.put(0, cfg);
  EEPROM.commit();
}

// ==== Pixel Mapping ====
uint16_t XY(uint8_t x, uint8_t y) {
  uint8_t tileX = x / TILE_WIDTH;
  uint8_t tileY = y / TILE_HEIGHT;
  uint8_t inX = x % TILE_WIDTH;
  uint8_t inY = y % TILE_HEIGHT;

  if (cfg.flipTiles) {
    inX = TILE_WIDTH - 1 - inX;
    inY = TILE_HEIGHT - 1 - inY;
  }

  if (cfg.serpentineTiles && tileY % 2 == 1)
    tileX = TILES_X - 1 - tileX;

  if (cfg.serpentineInTile && inY % 2 == 1)
    inX = TILE_WIDTH - 1 - inX;

  uint16_t indexInTile = inY * TILE_WIDTH + inX;
  uint16_t tileIndex = tileY * TILES_X + tileX;

  return tileIndex * (TILE_WIDTH * TILE_HEIGHT) + indexInTile;
}

uint32_t scaleColor(uint8_t r, uint8_t g, uint8_t b) {
  return strip.Color(r * cfg.brightness, g * cfg.brightness, b * cfg.brightness);
}

// ==== Asteroids Game Logic ====
void drawPixel(int x, int y, uint32_t color) {
  if (x < 0 || y < 0 || x >= MATRIX_WIDTH || y >= MATRIX_HEIGHT) return;
  strip.setPixelColor(XY(x, y), color);
}

void drawShip() {
  float rad = ship.angle;
  int cx = (int)ship.pos.x;
  int cy = (int)ship.pos.y;
  drawPixel(cx, cy, scaleColor(255, 255, 255));
  drawPixel(cx + cos(rad) * 1, cy + sin(rad) * 1, scaleColor(255, 255, 255));
  drawPixel(cx + cos(rad + 0.5f) * 1, cy + sin(rad + 0.5f) * 1, scaleColor(255, 255, 255));
}

void updateShip() {
  int x = nunchuck.readJoyX();
  int y = nunchuck.readJoyY();
  if (x < 100) ship.angle -= 0.1f;
  if (x > 150) ship.angle += 0.1f;
  if (y > 150) {
    ship.vel.x += cos(ship.angle) * 0.05f;
    ship.vel.y += sin(ship.angle) * 0.05f;
  }
  ship.pos.x += ship.vel.x;
  ship.pos.y += ship.vel.y;

  if (ship.pos.x < 0) ship.pos.x += MATRIX_WIDTH;
  if (ship.pos.x >= MATRIX_WIDTH) ship.pos.x -= MATRIX_WIDTH;
  if (ship.pos.y < 0) ship.pos.y += MATRIX_HEIGHT;
  if (ship.pos.y >= MATRIX_HEIGHT) ship.pos.y -= MATRIX_HEIGHT;
}

void fireBullet() {
  for (int i = 0; i < MAX_BULLETS; i++) {
    if (!bullets[i].active) {
      bullets[i].active = true;
      bullets[i].pos = ship.pos;
      bullets[i].vel.x = cos(ship.angle) * 0.8f;
      bullets[i].vel.y = sin(ship.angle) * 0.8f;
      break;
    }
  }
}

void updateBullets() {
  for (int i = 0; i < MAX_BULLETS; i++) {
    if (bullets[i].active) {
      bullets[i].pos.x += bullets[i].vel.x;
      bullets[i].pos.y += bullets[i].vel.y;
      if (bullets[i].pos.x < 0 || bullets[i].pos.x >= MATRIX_WIDTH || bullets[i].pos.y < 0 || bullets[i].pos.y >= MATRIX_HEIGHT) {
        bullets[i].active = false;
      } else {
        drawPixel((int)bullets[i].pos.x, (int)bullets[i].pos.y, scaleColor(255, 0, 0));
      }
    }
  }
}

void updateAsteroids() {
  for (int i = 0; i < MAX_ASTEROIDS; i++) {
    if (asteroids[i].active) {
      asteroids[i].pos.x += asteroids[i].vel.x;
      asteroids[i].pos.y += asteroids[i].vel.y;
      if (asteroids[i].pos.x < 0) asteroids[i].pos.x += MATRIX_WIDTH;
      if (asteroids[i].pos.x >= MATRIX_WIDTH) asteroids[i].pos.x -= MATRIX_WIDTH;
      if (asteroids[i].pos.y < 0) asteroids[i].pos.y += MATRIX_HEIGHT;
      if (asteroids[i].pos.y >= MATRIX_HEIGHT) asteroids[i].pos.y -= MATRIX_HEIGHT;
      drawPixel((int)asteroids[i].pos.x, (int)asteroids[i].pos.y, scaleColor(0, 255, 255));
    }
  }
}

void runAsteroidsGame() {
  strip.clear();
  nunchuck.readData();
  if (nunchuck.buttonZ || nunchuck.buttonC) fireBullet();
  updateShip();
  updateBullets();
  updateAsteroids();
  drawShip();
  strip.show();
}

// ==== LED Tests ====
void runTest() {
  for (int i = 0; i < NUM_LEDS; i++) {
    strip.clear();
    strip.setPixelColor(i, scaleColor(255, 255, 255));
    strip.show();
    delay(30);
  }
  strip.clear();
  strip.show();
}

void runRainbowTest(unsigned long durationMs = 10000) {
  unsigned long start = millis();
  uint8_t baseHue = 0;

  while (millis() - start < durationMs) {
    for (int y = 0; y < MATRIX_HEIGHT; y++) {
      for (int x = 0; x < MATRIX_WIDTH; x++) {
        uint8_t hue = baseHue + (x * 256 / MATRIX_WIDTH);
        CHSV hsv(hue, 255, 255);
        CRGB rgb;
        hsv2rgb_rainbow(hsv, rgb);
        strip.setPixelColor(XY(x, y), scaleColor(rgb.r, rgb.g, rgb.b));
      }
    }
    strip.show();
    delay(20);
    baseHue += 2;
  }
  strip.clear();
  strip.show();
}

// ==== Web Interface ====
void handleRoot() {
  String html = "<h1>Snake Config</h1>"
                "<form method='GET' action='/set'>"
                "Brightness (0.0–1.0): <input name='b' value='" + String(cfg.brightness) + "'><br>"
                "Difficulty (1-5): <input name='d' value='" + String(cfg.difficulty) + "'><br>"
                "<input type='submit' value='Save'></form>"
                "<br><a href='/test'>Run LED Test</a>"
                "<br><a href='/rainbow'>Run Rainbow Test</a>";
  server.send(200, "text/html", html);
}

void handleSet() {
  cfg.brightness = constrain(server.arg("b").toFloat(), 0.0f, 1.0f);
  cfg.difficulty = constrain(server.arg("d").toInt(), 1, 5);
  saveConfig();
  server.sendHeader("Location", "/");
  server.send(303);
}

void setupWeb() {
  server.on("/", handleRoot);
  server.on("/set", handleSet);
  server.on("/test", []() {
    server.send(200, "text/plain", "Running LED test...");
    runTest();
  });
  server.on("/rainbow", []() {
    server.send(200, "text/plain", "Running rainbow test...");
    runRainbowTest();
  });
  server.begin();
}

// ==== WiFi + OTA ====
void setupWiFi() {
  WiFi.mode(WIFI_STA);
  WiFiManager wm;
  if (!wm.autoConnect("SnakeGameAP", "snakesnek")) {
    Serial.println("WiFi failed, rebooting");
    ESP.restart();
  }
  Serial.println("Connected!");
}

void setupOTA() {
  ArduinoOTA.setHostname("SnakeESP32");
  ArduinoOTA.begin();
}

// ==== Main Setup ====
void setup() {
  Serial.begin(115200);
  loadConfig();
  strip.begin();
  strip.setBrightness(255);
  strip.clear();
  strip.show();

  Wire.begin();
  nunchuck.begin();
  nunchuck.update();

  setupWiFi();
  setupOTA();
  setupWeb();

  ship = { {MATRIX_WIDTH/2, MATRIX_HEIGHT/2}, {0, 0}, 0 };
}

void loop() {
  server.handleClient();
  ArduinoOTA.handle();
  runAsteroidsGame();
}

