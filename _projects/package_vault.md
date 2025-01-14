---
name: The Package Vault
tools: [Embedded C, Full Stack, Embedded Systems]
image: https://cdn.pixabay.com/photo/2024/06/26/23/36/package-8856091_640.png
description: A package theft prevention system project utilizing a smart lockbox with an extensive sensor array and a full stack software suite for device mangement and feature implementation.
pdf: Formal_Design_Document.pdf
---
# The Package Vault

### Tools
- **Embedded C**
- **Full Stack**
- **Embedded Systems**

### Description
A package theft prevention system project utilizing a smart lockbox with an extensive sensor array and a full-stack software suite for device management and feature implementation.

---

Since the rise of online shopping, many homes have experienced an increase in the number of packages delivered weekly. Unfortunately, this has been accompanied by a significant rise in porch package theft. Homeowners, often away for work or vacations, may only discover the theft of their packages upon returning home.

According to a study conducted by the Chamber of Commerce:
- **26% of consumers** have experienced package theft.
- The **average value** of stolen packages is $81.91.
- **38% of participants** believe that doorbell cameras do not effectively deter package thieves.

### Objective
To combat porch package theft, our project introduces a **smart vault system** designed to securely store packages upon delivery. This solution offers both **safety** and **convenience** for homeowners by ensuring deliveries are accessible only to authorized individuals.

---

### Key Features
- **Secure Access:** The vault unlocks only for authorized users, such as homeowners, delivery personnel, or any specified individuals.
- **Mobile Application:** Users can:
  - View a list of pending deliveries.
  - Set or reset the vault's unlock code.
  - Access live camera footage from the vault.

- **Dual-Chamber Design:**
  - **Top Chamber:** Packages are delivered here first.
  - **Bottom Chamber:** Packages are transferred here for secure storage once the top chamber is closed and relocked.
  - Access to the bottom chamber is restricted to the homeowner via a robust access control system.

---

### My Contribution
I was responsible for:
1. **Building the physical prototype:**
   - Constructed the dual-chamber lockbox with a tambour-style door separating the chambers.
   - Integrated motors for operating the tambour door.
2. **Embedded System Integration:**
   - Wrote the embedded C code to control physical interactions with the lockbox.
   - Integrated:
     - Electromagnetic locks to manage access to both chambers.
     - Magnetic sensors to detect door states (open/closed).

---

### Additional Resources
[Full Design Document](https://docs.google.com/document/d/1WqZcZflfH0FgMuc_nilx3sjoyTQBZSVSMhjYafap2CI/edit?usp=sharing)


---

### Code Snippet
Below is an example of the embedded C code I wrote to control the box:


```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_system.h"
#include "esp_log.h"
#include "driver/gpio.h"

//Comm pins with the sensor team
#define Top_Door_Pin GPIO_NUM_0
#define Bottom_Door_Pin GPIO_NUM_1
#define Middle_Door_Pin GPIO_NUM_2 

//Door magnet sensors
#define Top_Door_Sensor GPIO_NUM_4
#define Bottom_Door_Sensor GPIO_NUM_5
#define Mid_Door_Sensor GPIO_NUM_6

//Door locking bolt pins
#define Top_Door_Bolt GPIO_NUM_7
#define Bottom_Door_Bolt GPIO_NUM_8
#define Mid_Door_Bolt GPIO_NUM_21

//H-Bridge control pins for the motors
#define Motor_Pin_0 GPIO_NUM_10
#define Motor_Pin_1 GPIO_NUM_9
#define Motor_Pin_2 GPIO_NUM_20
#define Motor_Pin_3 GPIO_NUM_3

int open_top_door(){
    int mid_state = gpio_get_level(Mid_Door_Sensor);
    int bot_state = gpio_get_level(Bottom_Door_Sensor);
    int top_state;
    if(mid_state && bot_state){//Both the middle and bottom doors are closed so the top can open
        gpio_set_level(Top_Door_Bolt, 0);//open the top door bolt
        while((top_state = gpio_get_level(Top_Door_Sensor))){ //Keep the top door bolt open until the sensor detects it has been opened
            printf("Top door sensor %d\n", top_state);
            vTaskDelay(pdMS_TO_TICKS(100));
        }
        while(!(top_state = gpio_get_level(Top_Door_Sensor))){ //Keep the top door bolt open until the sensor detects it closed
        printf("Top door sensor %d\n", top_state);
            vTaskDelay(pdMS_TO_TICKS(100));
        }
        printf("Top door sensor %d\n", top_state);
        gpio_set_level(Top_Door_Bolt, 1);//close the top door bolt
        return 1;
    }
    return 0;
}

int open_mid_door() {
    gpio_set_level(Mid_Door_Bolt, 0); // Unlock the door
    printf("Unlocking MidDoor\n");
    vTaskDelay(pdMS_TO_TICKS(2000));
    gpio_set_level(Motor_Pin_0, 0);
    gpio_set_level(Motor_Pin_1, 1);
    gpio_set_level(Motor_Pin_2, 0);
    gpio_set_level(Motor_Pin_3, 1);
    vTaskDelay(pdMS_TO_TICKS(2000));
    int button_state = 0;
    while (!(button_state = gpio_get_level(Mid_Door_Sensor))) { // Keep the motors spinning until the middle door opens
        vTaskDelay(pdMS_TO_TICKS(50));
        printf("Opening Middle Door\n");
    }
    gpio_set_level(Motor_Pin_0, 1); //Left motor
    gpio_set_level(Motor_Pin_1, 0);
    gpio_set_level(Motor_Pin_2, 1); //Right Motor
    gpio_set_level(Motor_Pin_3, 0);
    vTaskDelay(pdMS_TO_TICKS(2000));
    int mid_state = 0;
    while (!(mid_state = gpio_get_level(Mid_Door_Sensor))) {
        vTaskDelay(pdMS_TO_TICKS(100));
        printf("Closing Middle Door\n");
    }
    gpio_set_level(Motor_Pin_0, 0);
    gpio_set_level(Motor_Pin_2, 0);
    gpio_set_level(Motor_Pin_1, 0);
    gpio_set_level(Motor_Pin_3, 0);
    gpio_set_level(Mid_Door_Bolt, 1); // Lock the door
    return 1;
}

int open_bot_door(){
    gpio_set_level(Bottom_Door_Bolt, 0);
    int bot_state = 0;
    while((bot_state = gpio_get_level(Bottom_Door_Sensor))){
        printf("Bottom door sensor %d\n", bot_state);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
    while(!(bot_state = gpio_get_level(Bottom_Door_Sensor))){
        printf("Bottom door sensor %d\n", bot_state);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
    gpio_set_level(Bottom_Door_Bolt, 1);
    printf("Bottom door sensor %d\n", bot_state);
    return 1;
}


void app_main() {
    // Configure GPIO pins as inputs
    esp_rom_gpio_pad_select_gpio(Top_Door_Pin);
    gpio_set_direction(Top_Door_Pin, GPIO_MODE_INPUT);

    esp_rom_gpio_pad_select_gpio(Bottom_Door_Pin);
    gpio_set_direction(Bottom_Door_Pin, GPIO_MODE_INPUT);

    esp_rom_gpio_pad_select_gpio(Bottom_Door_Sensor);
    gpio_set_direction(Bottom_Door_Sensor, GPIO_MODE_INPUT);

    esp_rom_gpio_pad_select_gpio(Mid_Door_Sensor);
    gpio_set_direction(Mid_Door_Sensor, GPIO_MODE_INPUT);

    esp_rom_gpio_pad_select_gpio(Top_Door_Sensor);
    gpio_set_direction(Top_Door_Sensor, GPIO_MODE_INPUT);

    //Configure GPIO pins as outputs
    esp_rom_gpio_pad_select_gpio(Middle_Door_Pin);
    gpio_set_direction(Middle_Door_Pin, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Top_Door_Bolt);
    gpio_set_direction(Top_Door_Bolt, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Bottom_Door_Bolt);
    gpio_set_direction(Bottom_Door_Bolt, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Mid_Door_Bolt);
    gpio_set_direction(Mid_Door_Bolt, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Motor_Pin_0);
    gpio_set_direction(Motor_Pin_0, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Motor_Pin_1);
    gpio_set_direction(Motor_Pin_1, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Motor_Pin_2);
    gpio_set_direction(Motor_Pin_2, GPIO_MODE_OUTPUT);

    esp_rom_gpio_pad_select_gpio(Motor_Pin_3);
    gpio_set_direction(Motor_Pin_3, GPIO_MODE_OUTPUT);

    int error_status;
    gpio_set_level(Top_Door_Bolt, 1);
    gpio_set_level(Bottom_Door_Bolt, 1);
    gpio_set_level(Mid_Door_Bolt, 1);
    gpio_set_level(Motor_Pin_0, 0);
    gpio_set_level(Motor_Pin_2, 0);
    gpio_set_level(Motor_Pin_1, 0);
    gpio_set_level(Motor_Pin_3, 0);
    gpio_set_level(Middle_Door_Pin, 0);
    while (1) {
        // Read requests from 
        int top_state = gpio_get_level(Top_Door_Pin);
        int bot_state = gpio_get_level(Bottom_Door_Pin);

        if (top_state) { //Sensor team is requesting for the top door to unlock
            printf("Top door unlock requested. Opening top door.......\n");
            error_status = open_top_door();
            printf("Top door closed. Cycling middle door.......\n");
            if(error_status != 0){
                gpio_set_level(Middle_Door_Pin, 1);
                error_status = open_mid_door();
                gpio_set_level(Middle_Door_Pin, 0);
            }
            
        }else if(bot_state){ //Sensor team is requesting for the bottom door to unlock
            printf("Bottom door unlock request\n");
            error_status = open_bot_door();
            printf("Bottom door closed.\n");
        }

        // Delay to avoid excessive polling
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```
