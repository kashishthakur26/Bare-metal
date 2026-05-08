# OS05B10 driver integration to RZ/G3E
This README provides guidance on 
1. Integrating the OS05B10 driver into the RZ/G3E kernel 
2. Extending its functionality with additional configurations, such as new image
   resolutions and frame rates.
3. How to use dynamic mode switch on G3E terminal.

## 1. Integrating the OS05B10 driver into the RZ/G3E kernel

### Usage:
  These source files will be useful if user wants to update source files direclty at respective folder. This method is preferable for debug purpose.

### Descreption:
- Below section `Source code details` tells the path of each source file in bitbake RZ/G3E VLP1.0.0 environment.  
- Source file are available at `source/`  
- Added driver source for FHD (1920x1080) @30FPS and @60FPS

### **Source code details**:  
-  Base path: `<Build_path>/build/tmp/work-shared/smarc-rzg3e/kernel-source`
-  Source files :   
    - `arch/arm64/boot/dts/renesas/rzg3e-smarc.dtsi`
    - `drivers/media/i2c/Kconfig`
    - `drivers/media/i2c/Makefile`
    - `drivers/media/i2c/os05b10.h`
    - `drivers/media/i2c/os05b10.c`
    - `drivers/media/platform/renesas/rzg2l-cru/rzg2l-core.c`
    - `drivers/media/platform/renesas/rzg2l-cru/rzg2l-video.c`

**Note**: Comapre the changes before replacing files

### Enable OS05B10 module in menuconfig:
-  Enable OS05B10 driver module in kernel. Open menu config.
   ```
   $ bitbake -c menuconfig virtual/kernel
   ```
-  Navigate to the driver module location.
   ```
   Device Drivers
   → Multimedia support
      → Media ancillary drivers
        → Camera sensor devices
          → OmniVision OS05B10 sensor support
   ```
-  Click 'Y' and save using keyboard arrows. The exit.

### Re-compile the image    
	```
    MACHINE=smarc-rzg3e bitbake -f -c compile linux-renesas
    MACHINE=smarc-rzg3e bitbake -f -c deploy linux-renesas
    MACHINE=smarc-rzg3e bitbake -f -c install linux-renesas
    MACHINE=smarc-rzg3e bitbake core-image-weston
	```
## 2 Adding additional configurations in OS05B10 driver
To add additional configurations in OS05B10 driver, user need to ready with
1. Image resolutions and fps
2. Register settings
3. HTS, VTS, exposure and pixel clock

Now we go step by step procedure to add new configuration in driver.
### 2. Add new regitser settings
1. Open `os05b10.h` file.
2. User can see the existing register settings.
   ```
   /* Register settings: FHD (1920x1080)@30fps */
   static struct regval os05b10_regs_fhd_30fps[] = {....}

   /* Register settings: VGA (640x480)@90fps */
   static struct regval os05b10_regs_vga_90fps[] = {....}
   ```
3. If the required image resolution register settings are not defined in this file, 
   add the necessary register settings by introducing a new global variable, following
   the same pattern as the existing one.
4. If the register settings for the required image resolution are already defined, 
   user only need to add the modifications related to the required fps.
5. Open `os05b10.c` file.
6. We call `new image resolution and fps setting` as **modes**.
7. User can see existing modes.
   ```
   static struct os05b10_mode supported_modes_10bit[] = 
   {
      {
         /* 1920x1080 30fps */
         .bus_fmt = MEDIA_BUS_FMT_SBGGR10_1X10,
         .width   = 1920,
         .height  = 1080,
         .fps     = 30,
         ...
         ...
      },
      {
         /* 640x480 90 fps */
         .bus_fmt = MEDIA_BUS_FMT_SBGGR10_1X10,
         .width   = 640,
         .height  = 480,
         .fps     = 90,
         ...
         ...
      },
   };
   ```
8. You can add a new mode for new resolution and update `VTS values based on FPS 
   requirement dynamiaclly`.
9.  Copy the existing one of the mode and edit the values.
10. For example if new mode is `1280x720 @240fps`. Then mode should be
    ```
    {
        /* 1280x720 240 fps */
        .bus_fmt = MEDIA_BUS_FMT_SBGGR10_1X10,
        .width   = 1280,
        .height  = 720,
        .fps     = 240,
        .vts     = 0x6D0,
        .hts     = 0x2B7,
        .exp     = 1700,
        .pixel_rate  = OS05B10_PIXEL_RATE_HD_240,
        .reg_list    = os05b10_regs_hd_240fps,
    },
    ```
11. `os05b10_regs_hd_240fps` should be in `os05b10.h` file.
12. Adjust `HTS, VTS, fps` and `exp` values as per the your `image resolution and 
    fps setting`.
13. Calculate `pixel_rate` using 
    ```
    pixel_rate >= HTS × VTS × FPS           
    ```
14. Configured `pixel_rate` should be more than calculated value. Use the existing 
    value if it works. Or adjust register settings for the pixel rate and create 
    new macro.
15. Below is the `pixel_rate` calculation for 210MHz.
    ```
    REF_CLK = 24MHz
    0x0326[7]   = 0 : pre_div0 = 1 (24MHz)
    0x0323[2:0] = 6 : pre_div  = 6 (6MHz)
    0x0324[1:0] = 0x1
    0x0325[7:0] = 0x3B : div_loop = 0x13B = 315 (1260MHz)
    0x0326[1:0] = 1 : pix_div  = 6 (210MHz)
    0x3661[7]   = 0 : dig_pixclk_div = 1 (210MHz)

    PIX_CLK = (REF_CLK / (pre_div0 * pre_div)) * (div_loop / (pix_div * dig_pixclk_div))
            = (24MHz/ (1 * 6)) * (315 / (6 * 1))
            = 210MHz

    ```
16. Update `div_loop` value to increase or decrease **PIX_CLK**.
16. As mentioned formula above, the user must adjust the register settings `0x0324 & 
    0x0325` to achieve the required `pixel_rate`.
17. Use this function call in `os05b10.c` if the new mode settings are identical to the  
    existing ones, with only register changes required.
    ```
    update_reg_settings(width, height, os05b10->cur_fps->val, \
                                os05b10->cur_mode->reg_list);
    ```
## 3. Dynamic mode switch on G3E terminal
1. Copy the scripts from git path  `source/scripts` to G3E `root` path.
2. Now initialize the `rzg3e-cru-os05b10 pipe line`.
   ```
   # ./init_os05b10.sh
   ```
3. Now configure `image resolution and fps`.
   ```
   # ./conf_os05b10.sh <image resolution> <fps>
   # ./conf_os05b10.sh 1920x1080 42
   ```
5. Now OS05B10 driver configured for `1920x1080 42 fps`.
6. User can check the fps of stream.
   ```
   # v4l2-ctl -d /dev/video0 --set-fmt-video=width=1920,height=1080,pixelformat=RG10 --stream-mmap --stream-count=1000
   ```
7. User can stream to HDMI display.
   ```
   gst-launch-1.0 v4l2src device=/dev/video0 ! 'video/x-raw,format=UYVY,width=1920,height=1080,framerate=60/1' ! vspmfilter dmabuf-use=true ! 'video/x-raw,format=NV12' ! fpsdisplaysink video-sink=waylandsink text-overlay=true
   ```
8. For more commands refer `doc/CHEAT_SHEET.txt`.
9. Added python script to test FHD (1920x1080) all FPS range from 1 to 80.  
   Python script location:  `source/scipt/vts_cal.py`
10. To run this python script, copy `conf_os05b10.sh, init_os05b10.sh and vts_cal.py` 
    into G3E and run `vts_cal.py`
    ```
    # python3 vts_cal.py
    ```
