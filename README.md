# How to improve the power consumption of LDC on BDR4261A
## Overview
**LDC** (low duty cycle mode) with Long Range DSSS PHY is a great feature for low power and long range. Regarding more deails on the theory of LDC, please refer to this article, [Low duty cycle mode](https://community.silabs.com/s/article/low-duty-cycle-mode?language=en_US).  
But in example project, there still a huge room to improve the power consumption. Here is the test result for **Long Preamble Duty Cycle** example.
### Board configuations
**4261A settings**    
- example:  Long Peamble Duty Cyle  
- PHY:      Long Range profile 490M 9.6Kbps  
- paramters:
    - Rx on time: 4167us
    - duty cycle period: 1.5s
### Power consumption of example
*4261A power consumption*  
- Average current - 40.67 uA 
<img src="images/4261A LDC power consumption.gif">   

## How to improve the power consumption
The formula of average current on one duty cycle is as follows.  

***Iavg = (Trx * Irx + Tsleep * Islepp) / Tperoid*** 
- ***Iavg*** - average current in one duty cycle
- ***Trx*** - the time of Rx state
- ***Irx*** - Rx current
- ***Tsleep*** - radio sleep time
- ***Isleep*** - sleep current
- ***Tperiod*** - the period time of LDC
### What can we do to improve the average current
- Reduce the sleep current
- Reduce the Rx on time
## Preparation
- Hardware   
    - BDR4261A (EFR32FG14P233F256GM48)  
- Software  
    - SDK version: V3.2.1
    - IDE: Simplicity studio V5
    - Example: Long Preamble Duty Cycle
    - Radio configuations: Long Range profile 490M 9.6Kbps
- The instrument required
    - Signal generator: E4432B
    - Spectrum analyzer: MS2692A 
    - Power analyzer: N6705C
## Creat a Long preamble duty cycle example project and configure
1.  Start Simplicity Studio V5
2.  Go to **Project**->**New**->**Silicon Labs Wizard...**
3.  Choose the board or device, select the SDK version V3.2.1, and then choose the **NEXT**.
4.  In the left **Technology type** tab, choose the **propriety**.
5.  In the right example list, choose **Felx(RAIL)-Long Preamble Duty Cycle** example, and then choose the **NEXT** and click **FINISH**.
6.  Find the **.slp** file in project and open it, choose the **CONFIGURATION TOOLS** and select **Radio Configurator**.
7. In the **General Settings** field, select the radio profile **Long Range Profile**,select the radio PHY **434MHz 9.6Kbps OQPSK DSSS SF8**.
8. Enable **Customized**, in the **Operational Frequency** field, fill the **Base Channel Frequency** to **490MHz**. 
9. In the **Other settings** field, make sure the **Long Range Mode** is **LR_9p6k**.
10. Radio configuations as the following figure.
<img src="images/radio configuation.png">
<img src="images/radio customized setting.png">  
11. Finally, type **Ctrl+S** to save the current configuations. the radio generator will automatically generate the relevant codes.  

## How to reduce the sleep current
EFR32xG14 sleep current in EM2 mode can be as low as **1.3uA**. Please refer to the datasheet, [ERF32FG14 datasheet](https://www.silabs.com/documents/public/data-sheets/efr32fg14-datasheet.pdf).
### Step 1: Uninstall LED driver
- Find the .slcp file and open it, In the **SOFTWARE COMPONENTS** field, search **LED** in software components. Uninstall the LED instance.
- Comment *"the sl_simple_led_instances.h"* head file in *app_init.c* file and *app_process.c* file.
- Find the LED control funtions in *app_init.c* file and *app_process.c* file and then comment it.  
### Step 2: Uninstall Button driver
Search **Button**, Uninstall the Button instance. 
### Step 3: Uninstall PTI component
Search **RAIL Utility**, uninstall the **RAIL Utility, Recommended** component of the PTI. 
### Step 4: Uninstall Graphics Library
Search the **Graphics**, uninstall the **GLIB Graphics Library** component.
### Step 5: Disable unused GPIO
Find the .pintool file and open it, change the unused GPIO mode to **None** as the fllowing figure.
<img src="images/PIN tool configuration.png">  

### Step 6: Disable Vcom
Search **board control** in software components, and click **configurate**. In the **Genteral** field, disable the **Enable Virtual COM UART** button.
### Step 7: Add TCXO control funcation
**Note**: If your board do not use TCXO, please ignore this step. I assume you use the 4261A board in here.
-  Find the ***sl_power_manager_hal_s0_s1.c*** in the path **gecko_sdk_3.2.1/platform/service/power_manager/src/sl_power_manager_hal_s0_s1.c** , and open it.
-  Try to modify the ***void sli_power_manager_handle_pre_deepsleep_operations(void)*** and ***void sli_power_manager_restore_high_freq_accuracy_clk(void)***, Simplicity Studio will pop up a warning box, click the **Make a Copy** to copy the this file from SDK library to project workspace.  
<img src="images/make a copy.png">
- Modify the funcations as following.  

```C
/***************************************************************************//**
 * Handle pre-deepsleep operations if any are necessary, like manually disabling
 * oscillators, change clock settings, etc.
 ******************************************************************************/
void sli_power_manager_handle_pre_deepsleep_operations(void)
  {
    if (is_hf_x_oscillator_used) {
        CMU_OscillatorEnable(cmuOsc_HFRCO, true, true);
        CMU_HFRCOBandSet(cmuHFRCOFreq_38M0Hz);
        CMU_ClockSelectSet(cmuClock_HF, cmuSelect_HFRCO);
        CMU_OscillatorEnable(cmuOsc_HFXO, false, false);
        sl_board_disable_oscillator(SL_BOARD_OSCILLATOR_TCXO);
    }
  } 
  
  /***************************************************************************//**
 * Handle post-sleep operations if any are necessary, like manually enabling
 * oscillators, change clock settings, etc.
 ******************************************************************************/
void sli_power_manager_restore_high_freq_accuracy_clk(void)
{
  if (is_hf_x_oscillator_used) {
      sl_board_enable_oscillator(SL_BOARD_OSCILLATOR_TCXO);
  }
} 
  ```
-  Include The TCXO control head file. Add the ***#include "sl_board_control.h"*** to this file. 
-  Add the ***sli_power_manager_private.h*** path to compiler. **Right click the project**->**properities**->**C/C++ build**->**setting**->**Tool setting**->**GNU ARM compiler**->**includes**, add the ***"${StudioSdkPath}/platform/service/power_manager/src"*** to path as the following figure.
<img src="images/include path.png">

### Step 8: Calculate the minimum of Rx on time
- Preamble detect need at least 40 bits. The formula is as follows.  
***Trx = 40 / 9.6 = 4.167 ms***
- Find ***the sl_duty_cycle_config.h*** in **config** catalog, and Modify the macro **DUTY_CYCLE_ON_TIME** to 4167.
```C
#define DUTY_CYCLE_ON_TIME      (4167)
```   
### step 9: Enable EM2 mode and disable button
Find the same file as above, and enable EM2 mode and disable button as following.
```c
#define DUTY_CYCLE_USE_LCD_BUTTON      0
#define DUTY_CYCLE_ALLOW_EM2           1
```

### Test the sleep current
-  Build the project and flash the hex file to target board.
-  Use the instrument to test sleep current. The sleep current as the following figure.
<img src="images/sleep current.gif">  

- Conclusion  
The sleep current is **1.94uA** after disable TCXO and unused GPIO. Due to there still some GPIO is working, so the result is in accord with the data from datasheet.  

## How to reduce the Rx on time
In the example peoject, the Rx on time is a fixed time even there is no any carrier in air. An appropriate approach is that radio will fast go to sleep when no any preamble is detected.
### Modity the timing of Preamble detection 
-  Add the macro to ***sl_duty_cycle_config.h*** file as following.
```C
#define DUTY_CYCLE_PERIOD        (1500000)
#define DUTY_CYCLE_OFF_TIME      (DUTY_CYCLE_PERIOD - DUTY_CYCLE_ON_TIME)
```
-  Add the macro to ***app_init.c*** file as following.
```C
// -----------------------------------------------------------------------------
//                              Macros and Typedefs
// -----------------------------------------------------------------------------
#define DUTY_CYCLY_SYNC_WORDS_TIME  1670
#define DUTY_CYCLY_MARGIN_TIME      1000
#define DUTY_CYCLY_SYNC_TIMEOUT     (DUTY_CYCLY_SYNC_WORDS_TIME + DUTY_CYCLE_PERIOD + DUTY_CYCLE_ON_TIME + DUTY_CYCLY_MARGIN_TIME)
```
-  Define the variable and configurate the timing for duty cycle as following.
```C
volatile RAIL_RxChannelHoppingConfigMultiMode_t duty_cycle_multi_mode_config = {
    .timingSense = 1600,
    .preambleSense = 2200,
    .syncDetect = DUTY_CYCLY_SYNC_TIMEOUT,
    .timingReSense = 1600,
    .status = 0,
}; 
/// Config for the correct timing of the dutycycle API
RAIL_RxDutyCycleConfig_t duty_cycle_config = {
  .delay = ((uint32_t) DUTY_CYCLE_OFF_TIME),
  .delayMode = RAIL_RX_CHANNEL_HOPPING_DELAY_MODE_STATIC,
  .mode = RAIL_RX_CHANNEL_HOPPING_MODE_MULTI_SENSE,
  .parameter =  (uint32_t) (void *) &duty_cycle_multi_mode_config
//  .parameter = ((uint32_t) DUTY_CYCLE_ON_TIME)
};
```
-  Please find the ***sl_duty_utilicy.c*** file in the path ***gecko_sdk_3.2.1/app/flex/component/rail/sl_duty_cycle_core*** and modify the funcation of calculating preamble bit length as following. Please select the **Make a Copy** when you try to modify this file.  
```C
uint16_t calculate_preamble_bit_length_from_time(const uint32_t bit_rate, RAIL_RxDutyCycleConfig_t * duty_cycle_config)
{
  float on_time = 0;
  float off_time = 0;
  float preamble_time = 0;
  float preamle_bit_length = 0;

  on_time = DUTY_CYCLE_ON_TIME;
  off_time = duty_cycle_config->delay;
  preamble_time = ((float)(PREAMBLE_PATTERN_LENGTH * PREAMBLE_PATTERN * PREAMBLE_OVERSAMPLING) * U_SEC) / bit_rate;

  app_assert(preamble_time < on_time, "Please modify the on time according to the bitrate!\n");

  while (1) {
    preamble_time = (off_time + (2 * on_time)) / 1000000;
    preamle_bit_length = (preamble_time * bit_rate);
    if (preamle_bit_length <= 50000) {
      break;
    }
    off_time = off_time - on_time;
  }

  if (((uint32_t)off_time) != DUTY_CYCLE_OFF_TIME) {
    app_log_warning("Duty Cycle Off time was changed to ensure stable working\n");
  }

  return preamle_bit_length;
}
```
### Modify the Radio PHY configuations
Please download the ***rail_config.c*** and ***rail_config.h*** file as the following link, and replace the original file in **Autogen** folder.  
[Optimized 9p6k radio configuation](radio_configuations/optimized_9p6k_configuation)
## Test conclusion
### Power Consumption
<img src="images/optimized power consumption.gif">

**Iavg = 19.036 uA**  
The power consumption of example is **40.674** uA, reduced power consumption by **53%** after optimizing. 
### Sensitivity
- Rx project: Optimized duty cycle
-  Use the [LR_Waveform_Generator](https://github.com/silabs-JimL/LR_WaveFormGenerator) to generate a waveform file and download the E4432B siganl generator. When the PER is 1%, the value at the monment represent the sensiticity.
-  Sensitivity is **-118.4 dBm** after optimizing.
-  The sensitivity of original example is **-119.1 dBm**.
-  Compared with original example,There is a **0.7 dBm** loss on sensitivity. But the loss is trivial and the optimization is well worth to do it.  
## FAQ
### Can we use the approach for other bitrate, for instance, 19.2kbps or 1.2kbps?
So far we only prodive the optimized parameters for **9.6kbps**. 1.2kbps and 19.2kbps will come soon.
### How to implement it on the board without TCXO ?
Just ignore the setp of modification for TXCO controlling.
## References
[KBA: Understanding DSSS Encoding and Decoding on EFR32 Devices](https://community.silabs.com/s/article/understanding-dsss-encoding-and-decoding-on-efr32-devices?language=en_US)  
[KBA: Low duty cycle mode](https://community.silabs.com/s/article/low-duty-cycle-mode?language=en_US)  
[UG460: EFR32 Series 1 Long Range Configuration Reference](https://www.silabs.com/documents/public/user-guides/ug460-efr32-series-1-long-range-configuration.pdf)  
[Datasheet: EFR32FG14 Flex Gecko Proprietary Protocol SoC Family Data Sheet](https://www.silabs.com/documents/public/data-sheets/efr32fg14-datasheet.pdf)  
