# GnomeCam

![IMG-20250317-WA0000](https://github.com/user-attachments/assets/215a28ad-4dea-4743-bede-54537215d692)

## Introduction
Ever wonder who knocked on your door and ran away?...  I thought of a fun little project to inconspicuously find out. 

As you'll have already figured out, a gnome, the seemingly innocent little man who guards our garden, 
will act as the conduit through which images are captured. Note, this project is purely for educational purposes.

## Material Requirements
### Hardware
Hardware | Purpose
--- | ---
ESP32 Wrover CAM Board, with a camera (mine came with the OV2640 camera) | Will act as the "brain" of the device.
Force Sensitive Material | Used trigger the GnomeCam to take a photo when someone steps on the doorstep. You can use any force sensing material - for example a Force FSR, a flex sensor, a piezo speaker, conductive foam, or velostat. I had many force sensitive resistors hanging around from a previous project so decided to use one of them.
Fixed resistor(s) | To be connected in series to the FSR so that collectively it works as a potential divider. Note, resistance used should reflect desired voltage change for weight of interest. In my case, I've used 2 x 1kOhm (as I didn't have a 2kOhm resistor handy)  - which results in a voltage change of ~2.5V for a load of 1.87 kilograms applied to the doorstep - which corresponds to a FSR value of 667 Ohms.
Battery | To power the device. I went with a 9V battery.
Step down converter| To step down the voltage from 5V to 3.3V (that used to operated the ESP32 microcontroller). Note, whilst the ESP32 microcontroller does have a built-in voltage regulator to convert 5V to 3.3V it does so using a linear converter - which is wasteful of energy and is subsequently not desirable for this low power application. I'd go with a standard buck converter if I didn't have a power supply module hanging around which I could use for free...
Jumper leads | To connect the aformentioned components in a circuit.

### Software 
1. Access to Arduino IDE
2. Access to Google Scripts, through a Google account. Note that, access via internet browser - there is no requirement to install anything.

## Design
### Circuit
#### Schematic

![circuit](https://github.com/user-attachments/assets/9d5d2917-130e-403b-9651-521d8e7ce8c0)


#### Power source
Gnomeo the spy needs to be powered by something. For Gnome food I opted for a 9V battery - as it's what I had lying around. I also had, lying around, a power supply module which regulates voltage - which I used to convert the 9V supply from the battery into the 3.3V needed to operate the ESP32. Note that, the power supply module is missing from the circuit diagram / schematic as I couldn't find it's symbol in the circuit diagram web editor. It should also be noted that you could also use a buck converter for this as I've highlighted in the material requirements section.
#### Wake up sensor
##### Introduction
Gnomeo needs to wake up when his FSR is pressed with a defined amount of force. We, as Gnome Gods, have the power to decide at what force that is. The way we do so is through a potential divider circuit... The equation for a potential divider is:

$$ V_{\text{out}} = V_{\text{in}} \times \frac{R_2}{R_1 + R_2} $$

Where:
- Vout is the output voltage across \( R_2 \) - which is what is converted into a digital value (HIGH or LOW), and which determines of Gnomeo should wake up or not.
- Vin is the input voltage - which is 3.3V provided from the ESP32.
- R1 and R2 are the resistances in the divider.

In the following subsections I'll describe how I determined and selected R1 and R2 - which affect Vout.

##### How to know what R2 is?
R2 is the FSR - which is effectively a variable resistor, where resistance decreases as pressure increases. Note that, as inferred from the table shown below, that relationship is not linear.

Whilst every FSR exhibits the same relationship between mass and resistance, to make sure my FSR was working as expected, and to be more sure of the load resistors to use in my potential divider, I thought I'd plot a mass versus resistance curve myself for my FSR. I did that by using my trusted multimeter - to measure the resistance across my FSR when different masses were applied to it. I repeated each mass for 3x and took the average resistance at each mass. Note that, some anomalies were excluded from the average. The results of that experiment are described in the table below....take note of the scientific objects which constitute the different masses - highly technical stuff.

| Object                                                  | Mass (kg) | Average resistance (kOhms) |
| ------------------------------------------------------- | --------- | -------------------------- |
| 9V Battery                                              | 0.046     | 10.5                       |
| Relay + Digit Screen + Remote Controller                | 0.03      | 15.7                       |
| Relay + Digit Screen + 9V Battery                       | 0.061     | 8.2                        |
| Relay + Digit Screen + Remote Controller + 9V Battery   | 0.076     | 5.3                        |
| Relay + Digit Screen + Pack of Cards                    | 0.112     | 3.3                        |
| Relay + Digit Screen + Allen Keys                       | 0.179     | 2.5                        |
| Relay + Digit Screen + Pack of Cards + Allen Keys       | 0.276     | 1.5                        |
| 2 x 1p Coin, Medal, 1 x Small Weight, 1 x Medium Weight | 1.87      | 0.7                        |
| 2 x 1p Coin, Medal, 1 x Small Weigh, 1 x Large Weight   | 3.025     | 0.5                        |

So, from that table, I am able to tell what R2 is at different masses. For example, at 1.87kg I know R2 takes a value of 0.7kOhms. That's great, now we only need to find a suitable value for R1, then we'll know Vout...

##### How to determine what value to use for R1?
By using the potential divider equation we can work out, for different values of R1, what will be the value of Vout. This is important, as it means we can impact the voltage we get out from our potential divider by using different a different value for R1 (sometimes referred to as the load resistor).

Ultimately, the reason for this potential divider circuit is to tell Gnomeo to wake up or not to wake up. Ultimately that is a binary decision - TRUE or FALSE, HIGH or LOW, 1 or 0. To integrate with the prebuilt ESP function for waking up from deep sleep this needs to be a digital signal of HIGH or LOW at the General Purpose Input Output (GPIO) pin to which Vout is connected... so, that leaves us with a little conundrum, as our outputs are analog and continous. How can we map them to either 1 or 0? Well, you could use a comparator. But, if you select your load resistance wisely, you won't have to.

For the ESP32 Wrover Module any voltage under 25% of operating voltage (so 0.825V) is assigned the value of 0, and any operating voltage over 75% of the operating voltage (so 2.475V) is assigned a value of 1. Between those regions behavior is more unpredictable. So, equipped with that knowledge, it'd be wise to build a system where voltage drops below 0.825V at the mass of interest. For me, I'm happy for Gnomeo to wake up when ~2kg is applied to my doorstep - which is more than a leaf but low enough to potentially take some photos of cats. Therefore, I set my value of R1 such that at 1.87kg Vout would be 0.825V. Equally, 2.4V wouldn't be passed until an object weighs more than about 70g. The value I set R1 to is 2kOhms. I don't have a 2kOhm resistor, so I used 2 x 1kOhms in series - which is the same.

#### Grounding voltage
The ground from the potential divider, the ESP32, and the power supply module should be connected.

### Code
#### Introduction
There are two main components to the code, which are:

* Microcontroller code - which is needed to process analog input from the FSR potential divider, to take the photo, and to send it (encoded), using Wi-Fi, to google scripts.
* Google Scripts code - which is needed to decode the image and to send it to the relevant Google Drive directory.

Both of these are explored more in the following subsections.

#### Microcontroller code
I'd be lying if I didn't say getting this code to work was tedious... I'm not (yet), although I do hope to be soon, proficient at coding in C / C++ - as such the concept of storing characters as pointers and char arrays went somewhat over my head. Whilst I did attempt to implement a solution where I appended to a char array - which had its memory dynamically attributed once an image was taken, this proved troublesome - likely due to my inexperience. Whilst I did also spend some time trying to learn C, I yearned a soluton, and so engineered one. Ultimately, it works. I'll stick a C / C++ book onto the reading list - I'm not done with it yet...

![](https://media1.giphy.com/media/v1.Y2lkPTc5MGI3NjExc2VtZjNpdTJ0M2JqZjl4cm9wYjhxajV3ZHprb2V2c3Y2ZTVnZjRzcyZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/rOsebqhlfCRby/giphy.gif)

Anyway, I don't want to bore you anymore, the code:

```
#include "esp_camera.h"
#include <WiFi.h>
#include "Base64_2.h"
#include <HTTPClient.h>

//
// WARNING!!! PSRAM IC required for UXGA resolution and high JPEG quality
//            Ensure ESP32 Wrover Module or other board with PSRAM is selected
//            Partial images will be transmitted if image exceeds buffer size
//
//            You must select partition scheme from the board menu that has at least 3MB APP space.
//            Face Recognition is DISABLED for ESP32 and ESP32-S2, because it takes up from 15
//            seconds to process single frame. Face Detection is ENABLED if PSRAM is enabled as well

// ===================
// Select camera model
// ===================
#define CAMERA_MODEL_WROVER_KIT // Has PSRAM
//#define CAMERA_MODEL_ESP_EYE  // Has PSRAM
//#define CAMERA_MODEL_ESP32S3_EYE // Has PSRAM
//#define CAMERA_MODEL_M5STACK_PSRAM // Has PSRAM
//#define CAMERA_MODEL_M5STACK_V2_PSRAM // M5Camera version B Has PSRAM
//#define CAMERA_MODEL_M5STACK_WIDE // Has PSRAM
//#define CAMERA_MODEL_M5STACK_ESP32CAM // No PSRAM
//#define CAMERA_MODEL_M5STACK_UNITCAM // No PSRAM
//#define CAMERA_MODEL_M5STACK_CAMS3_UNIT  // Has PSRAM
//#define CAMERA_MODEL_AI_THINKER // Has PSRAM
//#define CAMERA_MODEL_TTGO_T_JOURNAL // No PSRAM
//#define CAMERA_MODEL_XIAO_ESP32S3 // Has PSRAM
// ** Espressif Internal Boards **
//#define CAMERA_MODEL_ESP32_CAM_BOARD
//#define CAMERA_MODEL_ESP32S2_CAM_BOARD
//#define CAMERA_MODEL_ESP32S3_CAM_LCD
//#define CAMERA_MODEL_DFRobot_FireBeetle2_ESP32S3 // Has PSRAM
//#define CAMERA_MODEL_DFRobot_Romeo_ESP32S3 // Has PSRAM
#include "camera_pins.h"

// ===========================
// Enter your WiFi credentials
// ===========================
const char* ssid = "";
const char* password = "";

const int sensorPin = 32;
int sensorState = 0;

void setup() {
  
  pinMode(sensorPin, INPUT);

  Serial.begin(115200);
  Serial.setDebugOutput(true);
  Serial.println();

  //configure external wakeup
  esp_sleep_enable_ext0_wakeup((gpio_num_t) sensorPin, LOW);

  //delay to prevent multiple presses for the same person standing
  delay(5000);

  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sccb_sda = SIOD_GPIO_NUM;
  config.pin_sccb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.frame_size = FRAMESIZE_VGA;
  config.pixel_format = PIXFORMAT_JPEG;  // for streaming
  //config.pixel_format = PIXFORMAT_RGB565; // for face detection/recognition
  config.grab_mode = CAMERA_GRAB_WHEN_EMPTY;
  config.fb_location = CAMERA_FB_IN_PSRAM;
  config.jpeg_quality = 12;
  config.fb_count = 1;

  // if PSRAM IC present, init with UXGA resolution and higher JPEG quality
  //                      for larger pre-allocated frame buffer.

  if (psramFound()) {
    config.jpeg_quality = 10;
    config.fb_count = 2;
    config.grab_mode = CAMERA_GRAB_LATEST;
  } else {
    // Limit the frame size when PSRAM is not available
    config.fb_location = CAMERA_FB_IN_DRAM;
  }


  // camera init
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed with error 0x%x", err);
    return;
  }

  sensor_t *s = esp_camera_sensor_get();
  // initial sensors are flipped vertically and colors are a bit saturated
  if (s->id.PID == OV3660_PID) {
    s->set_vflip(s, 1);        // flip it back
    s->set_brightness(s, 1);   // up the brightness just a bit
    s->set_saturation(s, -2);  // lower the saturation
  }
  // drop down frame size for higher initial frame rate
  if (config.pixel_format == PIXFORMAT_JPEG) {
    s->set_framesize(s, FRAMESIZE_VGA);
  }

  camera_fb_t * fb = NULL;

  fb = esp_camera_fb_get();

  if(!fb) {
    Serial.println("Camera capture failed");
    return;
  }

  char *input = (char *)fb->buf;
  char output[base64_enc_len_2(3)];
  String imageFile ="";
  for (int i=0;i<fb->len;i++) {
    base64_encode_2(output, (input++), 3);
    if (i%3==0) imageFile += String(output);
  }
  Serial.print("length of fb is ");
  Serial.println(fb->len);

  Serial.print("length of image file is ");
  Serial.println(imageFile.length());

  WiFi.begin(ssid, password);
  WiFi.setSleep(false);

  Serial.print("WiFi connecting");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("");
  Serial.println("WiFi connected");

  HTTPClient http;

  http.begin("https://script.google.com/macros/s/[ENTER GOOGLE SCRIPTS DEPLOYMENT ID]");
  http.addHeader("Content-Type", "application/json");

  //note that, instead of using ***** can use any length string. It will be used a security feature - and removed before image is decoded in Google Scripts.
  int httpResponseCode = http.POST("****&%27(*" + imageFile);

  Serial.print("HTTP Response code: ");
  Serial.println(httpResponseCode);

  http.end();

  esp_deep_sleep_start();
  
}

void loop() {
}
```
Note that, you'll also need a number of supporting files to be saved in the same directory. Some of these are exist within the CameraWebServer example code - from where they were drawn - namely:
* app_httpd.cpp
* camera_index.h
* camera_pins.h
* ci.json

To encode each image, using base64 encoding, I modified some code from Adam Rudd. The only modifications I made was to append the suffix "2" to function names. Sounds like a strange modification right? I thought the same before I made it. But strangely, it seemed that retaining the naming convention caused conflicts with the HTTPClient library. Anyways, that code is:
* Base64_2.cpp
* Base64_2.h

### Google Scripts Code
The purpose of google scripts code is to decode the encoded image sent by the microcontroller (ESP32) and to upload it to the google drive - in your directory/folder of choice. The code is shown below.

```
function doPost(request) {

  var content_of_file = String(request.postData.contents);

  var GnomeCamCaptures_folder = DriveApp.getFolderById([ENTER YOUR FOLDER ID AS A STRING])

  try{

    //remove part that will corrupt the file. Note that, this part was added as a minor security feature - it can be any length string and is hardcoded into the microcontroller code. The idea being that people cannot send images to your deployment id and them be uploaded directly to your google drive. Each post request must have that identifier at the start (or at least in this case the correct length identifier!).
    content_of_file = content_of_file.slice(10);

    
    //js utilities method for decoding
    var decoded = Utilities.base64Decode(content_of_file);

    //create image object
    var blob = Utilities.newBlob(decoded, "image/jpeg", 'GnomeCapture.jpg'); // Please define date.

    //upload image object to directory of choice
    GnomeCamCaptures_folder.createFile(blob);

  }catch{

    //create error file if there is an error
    var file_error = GnomeCamCaptures_folder.createFile("error.txt", "error");

  }

}
```

### Physical Appearance
#### 3D Printing
I wanted my GnomeCam to actually look like a gnome. Instead of designing a Gnome that looked terrible I decided to see what open source stuff I could use. It was then when I stubled across this beautifully designed typical garden gnome from Sci3D on printables; which is originally from sci-nc on thingiverse and is licensed under the creative commons - attribution license. As describes on the printables page linked below this means its available for most use cases providing attribtion is given to the original artist.

https://www.printables.com/model/260908-garden-gnome/comments

For context, I have an Ender 3 V2 3D printer so was quite fortunate that I could print it myself. Although, I hate to admit how long it took to print... Anyway, I decided to print it with 0% infill as I wanted to fit my electronics inside the body. The end result can be seen below. Please ignore the terrible filming - at the time I thought it was "artsy".

https://github.com/user-attachments/assets/02d29062-2b0c-4d7b-9303-ba2a0f50467d

#### Creating Electronics Access Points
Whilst initially the plan was to mount all electronics in the gnome, for simplicity I decided to put only the ESP32 in the gnome and the other electronics in a plastic box I'd printed a while ago.

First step was creating a hole at the bottom so that my ESP32 could be inserted from underneath the gnome. I did this by drilling 4 holes for each vertex of a rectangle, before using a hacksaw blade to but inbetween the holes - creating a rectangular cutout. I dremel might be a cleaner and quicker way to achieve this, but the hacksaw blades were so cheap I couldn't justify the expense of a dremel.

Second step was drilling a hole for where the camera would sit - which happened to be at the belt area. I drilled two 8mm holes and the camera could see through fine.

Third step was drilling a hole in the plastic box - which wires connecting to the FSR could be passed through.

#### Painting
Before I painted Gnomeo I thought it'd be a good idea to get rid of the some of the sharp ridges from the 3D printing process. Though quite tedious, I think it did probably help in making the final product look more stone like. The sanded part (with part of the hat sliced off) can be seen below.

![IMG_0246](https://github.com/user-attachments/assets/98f63b2f-83e6-4b3d-88da-c22bdadb1328)

Next, I painted gnomeo. I used a stone paint and probably applied 4/5 layers. In hindsight I'd probably have painted the gnome white first, before applying stone paint. Regardless, I'd say it looks okay - as seen below.

![IMG_0254](https://github.com/user-attachments/assets/ed1eee0d-962e-490d-bc39-eb97e6fc52fc)

#### Mounting Electronics

Next step was mounting the ESP32 to the inside of the gnome. Conveniently it sat perched atop the legs - which meant all I had to do was sit in in place, hold it, then spray the ESP32 / leg interface with hot glue from a glue gun. It worked a treat! Worth mentioning, I'd label all your wires with masking tape before you do that. Else, its a bit tricky to tell which wire is for what. It doesn't look too bad from the image below.

![IMG_0259](https://github.com/user-attachments/assets/b4e59018-d4f4-4637-bb05-a97959deb98a)

After I mounted electronics I decided to connect the gnome to the enclosure which would contain the battery, voltage regulator and resistors for the potential divider circuit, using glue.

#### Using Heat Shrink Wrap for Exposed Wire
I used heat shrink wrap for exposed wires / soldered joints so that it'd be better suited for the outside. The result can be seen in the image below.

![IMG-20250317-WA0000](https://github.com/user-attachments/assets/98554956-8236-4631-b3ec-68b6aee29048)

### Future Improvements
Most of the issues I had towards the end of the project stemmed from the power source not being able to provide enough power and from electronics troubleshooting being a pain in the ass from me mounting the ESP32 inside the gnome without an easy access port for repairs.
* Test ESP32 with Arduino IDE at the maximum upload speed.... no wonder my debugging was taking an eternity.
* Use more stable power source. My 9V batteries didn't realiably work when their capacity was < 7.5V.
* Create easy access to ESP32 for repairs / troubleshooting.
* Prime the 3D print / print in white before applying stone paint.
