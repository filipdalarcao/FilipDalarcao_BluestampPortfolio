# Gesture Controlled Robot
I built a gesture controlled robot car, that moves when you move your hand. The Gyroscope on your hand connects with the car using bluetooth connected to an Arduino Uno. The Arduino connects to a motor controller which controls the motors.


| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Filip D | Leigh High School | Bio-Engineering | Incoming Sophomore


<img width="800" height="800" alt="IMG_0322" src="https://github.com/user-attachments/assets/af3b84a9-39f9-47a9-b3cf-f40174e9ad87" />

  
# Final Milestone


<iframe width="560" height="315" src="https://www.youtube.com/embed/JjjE3mH4_jA?si=P0NUh8IG-M7WUiRV" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

For my last milestone I had to finish the coding part of the project, and just the whole project. I made the code, and got all the components to work together, so now the project is complete. My biggest challenges at BSE were fixing a lot of code that wasn't working, and making the electrical part work. I learned that you have to keep trying and not give up, even if somethings not working, you have to keep retrying it. In the future I want to learn how to use Raspberry Pi, because from Bluestamp I learned arduino.



# Second Milestone


<iframe width="560" height="315" src="https://www.youtube.com/embed/Gyu8L54UIsw?si=HE-E1hzKvRhhemGA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

For my second milestone I had to finish all the electrical parts of my build. I wired together everything, and made sure it all works. So far the most surprising thing about the project was that the body is made of metal, so when I was wiring my components together, they would all short circuit. I overcame this by putting little plastic peices under all the components so the pins would not touch the metal. To finish my final milestone, I have to get the code working, and finish the project. 


# First Milestone



<iframe width="560" height="315" src="https://www.youtube.com/embed/1xphmux0Au4?si=WWS148a60P7QZxhY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

My project is a gesture controlled robot and my first milestone is building the body. The parts of the body are the wheels, the two metal body parts, the motors, and the H bridge. I put all those parts together, and now my milestone is done. The biggest challenge was screwing all the screws because some of them came out, and I had to screw in both levels of the chassis, and the wheel axels. But now everything is assembled and im ready to pursue my second milestone which is the electrical parts.


# Schematics 

<img width="800" height="800" alt="Screen Shot 2026-07-31 at 9 24 41 AM" src="https://github.com/user-attachments/assets/214a6e06-de9e-4c64-9d4d-4ec219b82f98" />



# Code


```c++
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>



int IN1 = 9;
int IN2 = 8;
int IN3 = 7;
int IN4 = 6;
int ENA = 12;
int ENB = 10;

const bool DEBUG = true;

const char* BLE_NAME = "GestureCar";


// so these are much smaller numbers than the Dabble version used.
const float TILT_THRESH  = 0.60;   // tilt needed before anything moves
const int DRIVE_SPEED = 250;   // tilt for full speed (~35 degrees)
const int TURN_SPEED = 160;

// Speed limits (0-255)
const int MAX_SPEED = 250;   
const int MIN_SPEED = 60;    

const bool INVERT_LEFT     = false;
const bool INVERT_RIGHT    = false;
const bool INVERT_THROTTLE = false;
const bool INVERT_STEERING = false;


const unsigned long PACKET_TIMEOUT_MS = 800;

#define NUS_SERVICE_UUID "6E400001-B5A3-F393-E0A9-E50E24DCCA9E"
#define NUS_RX_UUID      "6E400002-B5A3-F393-E0A9-E50E24DCCA9E"  // phone -> board
#define NUS_TX_UUID      "6E400003-B5A3-F393-E0A9-E50E24DCCA9E"  // board -> phone

volatile float  gAccelX = 0, gAccelY = 0;
volatile bool   gConnected = false;
volatile unsigned long gLastPacketMs = 0;

BLECharacteristic* txChar = nullptr;



uint8_t  pktBuf[24];
uint8_t  pktLen = 0;

float threshold = 0.05;


int expectedPacketLen(char type) {
  switch (type) {
    case 'A': return 15;  // accelerometer (x, y, z)
    case 'G': return 15;  // gyro
    case 'M': return 15;  // magnetometer
    case 'L': return 15;  // location
    case 'Q': return 19;  // quaternion
    case 'B': return 5;   // button
    case 'C': return 6;   // color
    default:  return -1;
  }
}

bool crcOk(const uint8_t* buf, uint8_t len) {
  uint8_t sum = 0;
  for (uint8_t i = 0; i < len - 1; i++) sum += buf[i];
  return (uint8_t)(~sum) == buf[len - 1];
}

float floatAt(const uint8_t* buf, int offset) {
  float f;
  memcpy(&f, buf + offset, 4);   
  return f;
}

void feedByte(uint8_t b) {
  
  if (pktLen == 0) {
    if (b != '!') return;
    pktBuf[pktLen++] = b;
    return;
  }

  pktBuf[pktLen++] = b;

  if (pktLen == 2) {
    if (expectedPacketLen((char)pktBuf[1]) < 0) {
      pktLen = 0;   
    }
    return;
  }

  int want = expectedPacketLen((char)pktBuf[1]);
  if (pktLen >= want) {
    if (crcOk(pktBuf, want) && pktBuf[1] == 'A') {
      gAccelX = floatAt(pktBuf, 2);
      gAccelY = floatAt(pktBuf, 6);
      
      gLastPacketMs = millis();
    }
    pktLen = 0;
  }

  if (pktLen >= sizeof(pktBuf)) pktLen = 0;   // safety
}


class RxCallbacks : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic* c) override {
    std::string v = c->getValue();
    for (size_t i = 0; i < v.length(); i++) feedByte((uint8_t)v[i]);
  }
};


class ServerCallbacks : public BLEServerCallbacks {
  void onConnect(BLEServer* s) override {
    gConnected = true;
    gLastPacketMs = millis();
  }
  void onDisconnect(BLEServer* s) override {
    gConnected = false;
    gAccelX = 0;
    gAccelY = 0;
    BLEDevice::startAdvertising();   // let it reconnect
  }
};



void driveMotor(int pwmPin, int in1, int in2, float value, bool invert) {
  if (invert) value = -value;

  if (fabs(value) < 0.01) {
    digitalWrite(in1, LOW);
    digitalWrite(in2, LOW);
    analogWrite(pwmPin, 0);
    return;
  }

  bool forward = (value > 0);
  digitalWrite(in1, forward ? HIGH : LOW);
  digitalWrite(in2, forward ? LOW  : HIGH);

  int speed = MIN_SPEED + (int)(fabs(value) * (MAX_SPEED - MIN_SPEED));
  analogWrite(pwmPin, constrain(speed, 0, 255));
}

void stopMotors() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
  analogWrite(ENA, 0);
  analogWrite(ENB, 0);
}

float axisToUnit(float v) {
  if(v > TILT_THRESH) return 1.0;
  if(v < -TILT_THRESH) return -1.0;

  return 0.0;
}



void setup() {
  Serial.begin(115200);

  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENB, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  stopMotors();

  BLEDevice::init(BLE_NAME);
  BLEServer* server = BLEDevice::createServer();
  server->setCallbacks(new ServerCallbacks());

  BLEService* service = server->createService(NUS_SERVICE_UUID);

  BLECharacteristic* rxChar = service->createCharacteristic(
      NUS_RX_UUID,
      BLECharacteristic::PROPERTY_WRITE | BLECharacteristic::PROPERTY_WRITE_NR);
  rxChar->setCallbacks(new RxCallbacks());

  txChar = service->createCharacteristic(
      NUS_TX_UUID, BLECharacteristic::PROPERTY_NOTIFY);
  txChar->addDescriptor(new BLE2902());

  service->start();

  BLEAdvertising* adv = BLEDevice::getAdvertising();
  adv->addServiceUUID(NUS_SERVICE_UUID);
  adv->setScanResponse(true);
  BLEDevice::startAdvertising();

  Serial.println("Advertising as GestureCar.");
  Serial.println("Bluefruit Connect -> connect -> Controller -> Accelerometer");
}

void loop() {
  bool live = gConnected &&
              (millis() - gLastPacketMs < PACKET_TIMEOUT_MS);
  
  if (!live) {
    stopMotors();
    delay(20);
    return;
  }

  float throttle = gAccelY;   // tilt forward/back
  float steering = gAccelX;   // tilt left/right

  static unsigned long lastDbg = 0;
  if(millis() - lastDbg > 300){
    lastDbg = millis();
    Serial.print(" x="); Serial.print(steering, 3);
    Serial.print(" y="); Serial.println(throttle, 3);
  }

  if (fabs(throttle) > threshold || fabs(steering) > threshold) {
    if (INVERT_THROTTLE) throttle = -throttle;
    if (INVERT_STEERING) steering = -steering;

    float left  = throttle + steering;
    float right = throttle - steering;

    float peak = max(fabs(left), fabs(right));
    if (peak > 1.0) {
      left  /= peak;
      right /= peak;
    }

 
    // Left Driver = ENA, IN1, IN2
    driveMotor(ENA, IN1, IN2, left,  INVERT_LEFT);
    // Right Driver = ENB, IN3, IN4
    driveMotor(ENB, IN3, IN4, right, INVERT_RIGHT);
  } else {
    stopMotors();
  }
  
  delay(10); 
}

```
<!-- Interactive Smartphone Tilt Controller Simulator -->
<div style="background-color: #1e1e1e; color: #ffffff; padding: 20px; border-radius: 12px; font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif; text-align: center; max-width: 500px; margin: 20px auto; box-shadow: 0 4px 12px rgba(0,0,0,0.3);">
  <h3 style="margin-top: 0; color: #4fc3f7;">Interactive Accelerometer Controller</h3>
  <p style="font-size: 0.9em; color: #cccccc; margin-bottom: 15px;">
    Open this portfolio on your phone, tap <b>Enable Motion Controls</b>, and tilt your smartphone to steer the simulated robot!
  </p>

  <canvas id="bseTiltCanvas" width="360" height="360" style="background: #121212; border: 2px solid #333333; border-radius: 8px; max-width: 100%; display: block; margin: 0 auto;"></canvas>

  <div style="margin-top: 15px;">
    <button id="bseMotionBtn" style="background: #00e676; color: #000000; border: none; padding: 10px 20px; font-size: 15px; font-weight: bold; border-radius: 6px; cursor: pointer; transition: 0.2s;">
      Enable Motion Controls
    </button>
  </div>

  <div style="display: flex; justify-content: space-around; margin-top: 15px; font-family: monospace; font-size: 0.85em; background: #2a2a2a; padding: 8px; border-radius: 6px;">
    <span>Pitch (Forward/Back): <b id="bsePitchVal" style="color: #4fc3f7;">0°</b></span>
    <span>Roll (Left/Right): <b id="bseRollVal" style="color: #ff4081;">0°</b></span>
  </div>
</div>

<script>
(function() {
  const canvas = document.getElementById('bseTiltCanvas');
  const ctx = canvas.getContext('2d');
  const btn = document.getElementById('bseMotionBtn');
  const pitchText = document.getElementById('bsePitchVal');
  const rollText = document.getElementById('bseRollVal');

  let robot = { x: canvas.width / 2, y: canvas.height / 2, angle: 0, speed: 0 };
  let pitch = 0, roll = 0;

  // Request motion permission (Required on iOS 13+)
  btn.addEventListener('click', async () => {
    if (typeof DeviceOrientationEvent !== 'undefined' && typeof DeviceOrientationEvent.requestPermission === 'function') {
      try {
        const response = await DeviceOrientationEvent.requestPermission();
        if (response === 'granted') {
          window.addEventListener('deviceorientation', handleOrientation);
          btn.innerText = "Motion Connected ✓";
          btn.style.background = "#333333";
          btn.style.color = "#00e676";
        } else {
          alert('Permission to access orientation was denied.');
        }
      } catch (e) {
        console.error(e);
      }
    } else {
      window.addEventListener('deviceorientation', handleOrientation);
      btn.innerText = "Motion Connected ✓";
      btn.style.background = "#333333";
      btn.style.color = "#00e676";
    }
  });

  function handleOrientation(event) {
    // Beta: Pitch (tilt forward/backward)
    // Gamma: Roll (tilt left/right)
    pitch = event.beta ? Math.max(-45, Math.min(45, event.beta)) : 0;
    roll = event.gamma ? Math.max(-45, Math.min(45, event.gamma)) : 0;

    pitchText.innerText = Math.round(pitch) + '°';
    rollText.innerText = Math.round(roll) + '°';
  }

  function update() {
    const deadzone = 3;
    let effPitch = Math.abs(pitch) > deadzone ? pitch : 0;
    let effRoll = Math.abs(roll) > deadzone ? roll : 0;

    robot.speed = (effPitch / 45) * 3;
    robot.angle += (effRoll / 45) * 0.05;

    robot.x += Math.sin(robot.angle) * robot.speed;
    robot.y -= Math.cos(robot.angle) * robot.speed;

    // Keep inside bounds
    robot.x = Math.max(15, Math.min(canvas.width - 15, robot.x));
    robot.y = Math.max(15, Math.min(canvas.height - 15, robot.y));
  }

  function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Draw Grid Pattern
    ctx.strokeStyle = '#222222';
    ctx.lineWidth = 1;
    for (let x = 0; x < canvas.width; x += 30) {
      ctx.beginPath(); ctx.moveTo(x, 0); ctx.lineTo(x, canvas.height); ctx.stroke();
    }
    for (let y = 0; y < canvas.height; y += 30) {
      ctx.beginPath(); ctx.moveTo(0, y); ctx.lineTo(canvas.width, y); ctx.stroke();
    }

    // Draw Robot Body
    ctx.save();
    ctx.translate(robot.x, robot.y);
    ctx.rotate(robot.angle);

    ctx.fillStyle = '#4fc3f7';
    ctx.fillRect(-15, -20, 30, 40);

    // Wheels
    ctx.fillStyle = '#ffffff';
    ctx.fillRect(-18, -15, 3, 10);
    ctx.fillRect(15, -15, 3, 10);
    ctx.fillRect(-18, 5, 3, 10);
    ctx.fillRect(15, 5, 3, 10);

    // Front Direction Pointer
    ctx.fillStyle = '#ff4081';
    ctx.beginPath();
    ctx.arc(0, -20, 5, 0, Math.PI * 2);
    ctx.fill();

    ctx.restore();
  }

  function loop() {
    update();
    draw();
    requestAnimationFrame(loop);
  }

  loop();
})();
</script>

# Bill of Materials

| **Part** | **Note** | **Price** | **Link** |
|:--:|:--:|:--:|:--:|
| Arduino Nano ESP32| Microcontroller | $19.30 | <a href="https://store-usa.arduino.cc/products/nano-esp32-with-headers?utm_source=google&utm_medium=cpc&utm_campaign=US-Pmax&gad_source=1&gad_campaignid=21317508903&gbraid=0AAAAACbEa8495Cjbem1beiV2598e-NST7&gclid=CjwKCAjwj7HTBhBiEiwA8s35OqpQHvwXaUFhnHCSP6Uqrj1pd16D5lqWVEWMdvKmvLJBcFI_ZPnwOhoConYQAvD_BwE"> Link </a> |
| L298N Motor Drive Controller | Motor Driver | $6.99 | <a href="https://www.amazon.com/dp/B014KMHSW6?lv=shuf&channelId=500&plpRedirect=mhFallback"> Link </a> |
| Yellow TT Motor | Motor | $4 | <a href="[https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/](https://www.amazon.com/dp/B07L881GXZ?lv=shuf&channelId=500&plpRedirect=mhFallback)"> Link </a> |
| 4WD Omni-wheel Robot Car Metal Chassis | Body of Robot | $39.99 | <a href="[https://www.amazon.com/Arduino-A000066-ARDUINO-UNO-R3/dp/B008GRTSV6/](https://tscinbuny.com/products/tscinbuny-4wd-omni-wheel-robot-car-metal-chassis-for-arduino-robotic-project?srsltid=AfmBOop6Gvt8zAMGuKzlqsgk_bIsCMRnhd3i3FTzRNGI2_1eoBUjmyByTyw)"> Link </a> |

# References

- [H Bridge](https://www.circuitbread.com/ee-faq/how-does-an-h-bridge-work)
- [Bluefruit](https://www.instructables.com/Wireless-Serial-Communication-Using-Bluefruit/)
- [Arduino](https://docs.arduino.cc/learn/starting-guide/getting-started-arduino/)


