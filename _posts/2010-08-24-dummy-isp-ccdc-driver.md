---
layout: article
title: dummy isp ccdc driver
categories: unfinished isp-ccdc
updated_at: 2010-08-24
---
Goal
====

Create a simple dummy driver (called `fsr172x`) for the OMAP3530 that bypasses almost all extra functionality just to get the `RAW` data from the 12 data lines to a `v4l2buf`.

Overview
----

  * `fsr172x` is a platform-independent v4l2-**slave** device
    * controls the sensor device
    * interfaces with `v4l2`
    * is unaware of `isp`
  * `omap34xxcam` is the platform-dependent v4l2-**master**
    * controls the `isp`
    * interfaces with `v4l2`
  * `v4l2` provides a common framework which the application may use
  * `isp` image support package?
  * `ispccdc`
    * controls the `ccdc`
  * `fidcap` is a dummy capture application which utilizes `v4l2`
    * `fidcap` is designed specifically for use with the `fsr172x` and therefore is a bad example
  * the **master** and **slave** have no knowledge of each other's implementations
  * a platform "driver" such as `board-overo-camera` (Gumstix Overo) or `board-omap3beagle-camera.c` (BeagleBoard) connects the two together..

**Note**: The file contents shown are not complete but rather snippets that give content to the changes.
**Note**: The FSR172X is a stripped down version of the MT9T111. I recommend viewing the [V4L2 Example Capture application](http://v4l2spec.bytesex.org/spec/capture-example.html) as well.

Pre-reqs
----

  * Get a copy of [`linux-omap-psp`](2010-08-12-omap-dvsdk-on-openembedded.md) in your `~/`
  * Review [this v4l2 struct](http://www.videolan.org/developers/vlc/doc/doxygen/html/structv4l2__format.html)

Hacking V4L2
====

Our dummy driver will use a special formats for

  * standard: `FSRX` - as opposed to `NTSC`, `PAL`, etc
  * colorspace: `FSR` - as opposed to `SRGB`

These are created from `enum`s and `#define`s that are completely arbitrary.
The caveat is that any application or driver that depends on these settings
must include this special modified version of the of `videodev2.h`.

`drivers/media/video/isp/isp.c`
-----------

This is a patch showing changes I've made for the fsr172x. Once I have this figured out I'll refactor it down into understandable chunks

    diff --git a/drivers/media/video/isp/isp.c b/drivers/media/video/isp/isp.c
    index 29dd005..2369f54 100644
    --- a/drivers/media/video/isp/isp.c
    +++ b/drivers/media/video/isp/isp.c
    @@ -930,7 +930,7 @@ static irqreturn_t omap34xx_isp_isr(int irq, void *_pdev)
        return IRQ_HANDLED;

      spin_lock_irqsave(&bufs->lock, flags);
    - wait_hs_vs = bufs->wait_hs_vs;
    + wait_hs_vs = 0; //bufs->wait_hs_vs; For FSR
      if (irqstatus & HS_VS) {
        if (bufs->wait_hs_vs) {
          bufs->wait_hs_vs--;
    @@ -978,12 +978,23 @@ static irqreturn_t omap34xx_isp_isr(int irq, void *_pdev)
      }

      if (irqstatus & CCDC_VD0) {
    +   DPRINTK_ISPCTRL("VD0 interupt\n");    
        if (isp->pipeline.pix.field == V4L2_FIELD_INTERLACED) {
          /* Skip even fields, and process only odd fields */
          if (isp->current_field != 0)
            if (RAW_CAPTURE(isp))
              isp_buf_process(dev, bufs);
        }
    +   else // For FSR - progressive scan, process every field
    +   {
    +     //DPRINTK_ISPCTRL("Not interlaced\n");
    +     if (RAW_CAPTURE(isp))
    +     {
    +       DPRINTK_ISPCTRL("Processing buff\n");
    +       isp_buf_process(dev, bufs);
    +     }
    +   }
    +   
        if (!ispccdc_busy(&isp->isp_ccdc))
          ispccdc_config_shadow_registers(&isp->isp_ccdc);
      }
    @@ -1363,7 +1374,10 @@ static void isp_set_buf(struct device *dev, struct isp_buf *buf)
          && is_ispresizer_enabled())
        ispresizer_set_outaddr(&isp->isp_res, buf->isp_addr);
      else if (isp->pipeline.modules & OMAP_ISP_CCDC)
    + {
    +   printk("SDR Address set to 0x%08X\n",buf->isp_addr);
        ispccdc_set_outaddr(&isp->isp_ccdc, buf->isp_addr);
    + }

     }

    @@ -1412,7 +1426,10 @@ static int isp_try_pipeline(struct device *dev,
        pipe->prv_out = PREVIEW_MEM;
        pipe->rsz_in = RSZ_MEM_YUV;
      } else {
    +    printk("ARRIVED IN: ISP_CCDC\n");
        pipe->modules = OMAP_ISP_CCDC;
    +   // TODO if this is set in fsr172x, we shouldn't need this here
    +   //pix_input->pixelformat = V4L2_PIX_FMT_FSR172X; // Added for FSR 
        if (pix_input->pixelformat == V4L2_PIX_FMT_SGRBG10 ||
            pix_input->pixelformat == V4L2_PIX_FMT_SGRBG10DPCM8 ||
            pix_input->pixelformat == V4L2_PIX_FMT_SRGGB10 ||
    @@ -1431,12 +1448,17 @@ static int isp_try_pipeline(struct device *dev,
        } else if (pix_input->pixelformat == V4L2_PIX_FMT_YUYV ||
             pix_input->pixelformat == V4L2_PIX_FMT_UYVY) {
          if (isp->bt656ifen)
    -       pipe->ccdc_in = CCDC_YUV_BT;
    +       pipe->ccdc_in = CCDC_YUV_BT;
          else
    -       pipe->ccdc_in = CCDC_YUV_SYNC;
    +       pipe->ccdc_in = CCDC_YUV_SYNC;
          pipe->ccdc_out = CCDC_OTHERS_MEM;
    -   } else
    +   } else if (pix_input->pixelformat == V4L2_PIX_FMT_FSR172X) { // Added for FSR
    +      printk("ARRIVED IN: PIPELINE FSR\n");
    +     pipe->ccdc_in = CCDC_RAW_FSR;
    +     pipe->ccdc_out = CCDC_FSR_MEM;
    +    } else {
          return -EINVAL;
    +    }
      }

      if (pipe->modules & OMAP_ISP_CCDC) {
    @@ -1652,7 +1674,8 @@ static int isp_buf_process(struct device *dev, struct isp_bufs *bufs)
      if (ISP_BUFS_IS_EMPTY(bufs))
        goto out;

    - if (RAW_CAPTURE(isp) && ispccdc_sbl_wait_idle(&isp->isp_ccdc, 1000)) {
    +  // We keep getting wait errors... this seems to fix the problem (originally 1000)
    + if (RAW_CAPTURE(isp) && ispccdc_sbl_wait_idle(&isp->isp_ccdc, 2e6)) {
        dev_err(dev, "ccdc %d won't become idle!\n",
               RAW_CAPTURE(isp));
        goto out;


`drivers/media/video/isp/ispccdc.c`
----------

    diff --git a/drivers/media/video/isp/ispccdc.c b/drivers/media/video/isp/ispccdc.c
    index 137a5e6..df7c15d 100644
    --- a/drivers/media/video/isp/ispccdc.c
    +++ b/drivers/media/video/isp/ispccdc.c
    @@ -631,6 +631,23 @@ static int ispccdc_config_datapath(struct isp_ccdc_device *isp_ccdc,
        ispccdc_config_vp(isp_ccdc, vpcfg);
        ispccdc_enable_vp(isp_ccdc, 1);
        break;
    +
    +  case CCDC_FSR_MEM:
    +    printk("ARRIVED IN: CCDC_FSR_MEM\n");
    +   syn_mode &= ~ISPCCDC_SYN_MODE_VP2SDR;
    +   syn_mode &= ~ISPCCDC_SYN_MODE_SDR2RSZ;
    +   syn_mode |= ISPCCDC_SYN_MODE_WEN;
    +   syn_mode &= ~ISPCCDC_SYN_MODE_EXWEN;
    +    isp_reg_and(isp_ccdc->dev, OMAP3_ISP_IOMEM_CCDC,
    +        ISPCCDC_CFG, ~ISPCCDC_CFG_WENLOG);
    +
    +    // TODO aren't these 0 by default?
    +   vpcfg.bitshift_sel = 0;
    +   vpcfg.freq_sel = 0;
    +   ispccdc_config_vp(isp_ccdc, vpcfg);
    +   ispccdc_enable_vp(isp_ccdc, 0);
    +    break;
    +
      default:
        DPRINTK_ISPCCDC("ISP_ERR: Wrong CCDC Output\n");
        return -EINVAL;
    @@ -700,6 +717,33 @@ static int ispccdc_config_datapath(struct isp_ccdc_device *isp_ccdc,
        blkcfg.dcsubval = 0;
        ispccdc_config_black_clamp(isp_ccdc, blkcfg);
        break;
    +  case CCDC_RAW_FSR: // what values do we need here?
    +    printk("ARRIVED IN: CCDC_RAW_FSR\n");
    +    syncif.ccdc_mastermode = 1; // p1461 1 ==hs/vs output
    +    syncif.datapol = 0; // 0 == normal, not one's complement
    +    syncif.datsz = DAT12; // 12 lines
    +    syncif.fldmode = 0; // 0 == progressive scan, not interlaced
    +    syncif.fldout = 0; // (cam_fld) not used when 0 == fldmode
    +    syncif.fldpol = 0; // not used when 0 == fldmode
    +    syncif.fldstat = 0; // not used when 0 == fldmode, otherwise marks current frame as odd/even
    +    syncif.hdpol = 0; // cam_hs polarity 0 == positive
    +    syncif.ipmod = RAW; // aka inpmod p1553 should be 0 for raw
    +    syncif.vdpol = 0; // cam_vs polarity 0 == positive
    +    syncif.bt_r656_en = 0; // 1 == ITU enabled
    +
    +    // These should next four should always be set when mastermode is enabled
    +    syncif.hs_width = 0; // aka hd_vd_wid p1554
    +    syncif.vs_width = 0; // aka hd_vd_wid
    +    syncif.ppln = (u8) 0x7D01; // 32001 Maybe should be 16001(?) p1555 pixels per line
    +    syncif.hlprf = 128; // half line per field or frame
    +
    +    // isp modules
    +    ispccdc_config_imgattr(isp_ccdc, 0); // mosiac filter??, probably should be all 0's
    +    ispccdc_config_sync_if(isp_ccdc, syncif); // commit these values to the registers
    +
    +    // black clamp
    +    blkcfg.dcsubval = 0; // black clamp substraction value, probably should be 0
    +    ispccdc_config_black_clamp(isp_ccdc, blkcfg);
      case CCDC_OTHERS:
        break;
      default:
    @@ -1202,6 +1246,8 @@ int ispccdc_try_pipeline(struct isp_ccdc_device *isp_ccdc,
      pipe->ccdc_in_v_st = 0;
      pipe->ccdc_out_w = pipe->ccdc_in_w;
      pipe->ccdc_out_h = pipe->ccdc_in_h;
    + 
    + printk("Out pixel height %d\n",pipe->ccdc_out_h);

      if (!isp_ccdc->refmt_en
          && pipe->ccdc_out != CCDC_OTHERS_MEM
    @@ -1348,6 +1394,7 @@ int ispccdc_s_pipeline(struct isp_ccdc_device *isp_ccdc,
              OMAP3_ISP_IOMEM_CCDC,
              ISPCCDC_VDINT);
        } else {
    +       printk("Arrived in vp out mem\n");
          ispccdc_config_outlineoffset(isp_ccdc,
              pipe->ccdc_out_w * 2, EVENEVEN, 1);
          ispccdc_config_outlineoffset(isp_ccdc,
    @@ -1367,6 +1414,7 @@ int ispccdc_s_pipeline(struct isp_ccdc_device *isp_ccdc,
        }

      } else if (pipe->ccdc_out == CCDC_OTHERS_VP_MEM) {
    +   printk("Arrived in vp out mem\n");
        isp_reg_writel(isp_ccdc->dev,
                 (pipe->ccdc_in_h_st << ISPCCDC_FMT_HORZ_FMTSPH_SHIFT) |
                 ((pipe->ccdc_in_w - pipe->ccdc_in_h_st) <<


`drivers/media/video/isp/ispccdc.h`
----------

The FSR uses a type of `RAW` capture and has particular memory requirements not currently supported by the `ISP`.
These new `enum`s allow us to cleanly create new conditional branches in the `ISP`/`CCDC` code.

    /* Enumeration constants for CCDC input output format */
    enum ccdc_input {
      CCDC_RAW_GRBG,
      CCDC_RAW_RGGB,
      CCDC_RAW_BGGR,
      CCDC_RAW_GBRG,
      CCDC_RAW_FSR, // FSR
      CCDC_YUV_SYNC,
      CCDC_YUV_BT,
      CCDC_OTHERS
    };

    enum ccdc_output {
      CCDC_YUV_RSZ,
      CCDC_YUV_MEM_RSZ,
      CCDC_FSR_MEM, // FSR
      CCDC_OTHERS_VP,
      CCDC_OTHERS_MEM,
      CCDC_OTHERS_VP_MEM
    };


`include/linux/videodev2.h`
-----------

Note: Although these `#define`s are arbitrary, they must match between the file `include`d
with the driver and the file `include`d in the application.

    // Our own special colorspace goes in a section like this
    enum v4l2_colorspace {
      /* ITU-R 601 -- broadcast NTSC/PAL */
      V4L2_COLORSPACE_SMPTE170M     = 1,

      /* 1125-Line (US) HDTV */
      V4L2_COLORSPACE_SMPTE240M     = 2,

      //
      // SEVERAL LINES SKIPPED FOR BREVITY IN THIS EXAMPLE
      //

      /* For RGB colourspaces, this is probably a good start. */
      V4L2_COLORSPACE_SRGB          = 8,

      /* For colourspaces not in the normal range (FSR172X). */
      V4L2_COLORSPACE_FSR172X       = 9,
    };


    // Our own special pixel format goes in a section like this
    /*
     *  V I D E O   I M A G E   F O R M A T
     */

    //
    // SEVERAL LINES SKIPPED FOR BREVITY IN THIS EXAMPLE
    //

    /*  Vendor-specific formats   */
    #define V4L2_PIX_FMT_WNVA     v4l2_fourcc('W', 'N', 'V', 'A') /* Winnov hw compress */
    #define V4L2_PIX_FMT_SN9C10X  v4l2_fourcc('S', '9', '1', '0') /* SN9C10x compression */

    //
    // SEVERAL LINES SKIPPED FOR BREVITY IN THIS EXAMPLE
    //

    #define V4L2_PIX_FMT_STV0680  v4l2_fourcc('S', '6', '8', '0') /* stv0680 bayer */
    #define V4L2_PIX_FMT_FSR172X  v4l2_fourcc('F', 'S', 'R', '0') /* 12bit raw fsr172x */

`include/linux/v4l2-int-device.h`
-----------

Nothing is changed here, but it's worth noting that we'll be using the predefined `RAW` mode...

    /* Slave interface type. */
    enum v4l2_if_type {
      /*
       * Parallel 8-, 10- or 12-bit interface, used by for example
       * on certain image sensors.
       */
      V4L2_IF_TYPE_BT656,
      V4L2_IF_TYPE_YCbCr,
      V4L2_IF_TYPE_RAW,
    };

... and its associated struct

    struct v4l2_if_type_raw {
      /*
       * 0: Frame begins when vsync is high.
       * 1: Frame begins when vsync changes from low to high.
       */
      unsigned frame_start_on_rising_vs:1;
      /* Use Bt synchronisation codes for sync correction. */
      unsigned bt_sync_correct:1;
      /* Swap every two adjacent image data elements. */
      unsigned swap:1;
      /* Inverted latch clock polarity from slave. */
      unsigned latch_clk_inv:1;
      /* Hs polarity. 0 is active high, 1 active low. */
      unsigned nobt_hs_inv:1;
      /* Vs polarity. 0 is active high, 1 active low. */
      unsigned nobt_vs_inv:1;
      /* Minimum accepted bus clock for slave (in Hz). */
      u32 clock_min;
      /* Maximum accepted bus clock for slave. */
      u32 clock_max;
      /*
       * Current wish of the slave. May only change in response to
       * ioctls that affect image capture.
       */
      u32 clock_curr;
    };

If we were to create our own, we would need to modify this v4l2 struct:

    struct v4l2_ifparm {
      enum v4l2_if_type if_type;
      union {
        struct v4l2_if_type_bt656 bt656;
        struct v4l2_if_type_ycbcr ycbcr;
        struct v4l2_if_type_raw raw;
      } u;
    };

In our driver code we will access these settings as `v4l2_ifparm`.

`arch/arm/mach-omap2/board-overo-camera.c`
-----------

This board file and its header are the exact same as `board-omap3beagle-camera` with the exception that every occurrance of *beagle* has been replaced with *overo* and every occurance of *mt9t111* has been replaced with *fsr172x* and the following diff.

    #include "board-overo-camera.h"             | #include "board-omap3beagle-camera.h"
                        |
    //    if (regulator_is_enabled(beagle_fsr172x_1_8v1 |     if (regulator_is_enabled(beagle_fsr172x_1_8v1
    //      regulator_disable(beagle_fsr172x_1_8v |       regulator_disable(beagle_fsr172x_1_8v
    //    if (regulator_is_enabled(beagle_fsr172x_1_8v2 |     if (regulator_is_enabled(beagle_fsr172x_1_8v2
    //      regulator_disable(beagle_fsr172x_1_8v |       regulator_disable(beagle_fsr172x_1_8v
    //    gpio_set_value(LEOPARD_RESET_GPIO, 0);        |     gpio_set_value(LEOPARD_RESET_GPIO, 0);
    //    regulator_enable(beagle_fsr172x_1_8v1);       |     regulator_enable(beagle_fsr172x_1_8v1);
    //    regulator_enable(beagle_fsr172x_1_8v2);       |     regulator_enable(beagle_fsr172x_1_8v2);
    //    gpio_set_value(LEOPARD_RESET_GPIO, 1);        |     gpio_set_value(LEOPARD_RESET_GPIO, 1);
    #if 0                   <
    #endif                    |
    #if 0                   <
    #endif                    |


Again, we'll just use `RAW`, but if we had created our own `FSR` standard, we would need to add to this list:

    static struct v4l2_ifparm fsr172x_ifparm_s = {
    #if 1
      .if_type = V4L2_IF_TYPE_RAW,
      .u   = {
        .raw = {
          .frame_start_on_rising_vs = 1,
          .bt_sync_correct  = 0,
          .swap     = 0,
          .latch_clk_inv    = 0,
          .nobt_hs_inv    = 0,  /* active high */
          .nobt_vs_inv    = 0,  /* active high */
          .clock_min    = FSR172X_CLK_MIN,
          .clock_max    = FSR172X_CLK_MAX,
        },
      },
    #else
      .if_type = V4L2_IF_TYPE_YCbCr,
      // REMOVED FOR BREVITY
      },
    #endif
    };

scratchspace
=====

Linux Kernel vs TRM
-------

  * omap34xxcam_hw_config.u.sensor.sensor_isp -> ???
  * omap34xxcam_hw_config.wait_hs_vs -> ???
  * isp_interface_config.u.par.par_bridge -> ???
  * omap34xxcam_videodev.cam.isp -> ???
    * v4l2_int_device->u.slave->master->priv -> ???

where stuff is
-------

  * v4l2_ifparm: v4l2-int-device
    * fsr172x.c
    * board-overo-camera.c

  * omap34xxcam_hw_config: omap34xxcam
    * board-overo-camera.c

which v4l2 calls must we implement?
-------

  * ???

what is this mess?
------

  * regulator_put - part of `include/linux/regulator/consumer.h` - for future compatibility. Does nothing as of right now. Just a stubbed out API.
  * dev_pm_ops - power management for suspending, resuming, etc
  * omap34xxcam is a **master**
  * fsr172x is a **slave**

`drivers/media/video/fsr172x.c`
-----------

    // Creating a map between VIDIOC_X_YZ and our internal functions
    static struct v4l2_int_ioctl_desc fsr172x_ioctl_desc[] = {
      { .num = vidioc_int_enum_framesizes_num,
        .func = (v4l2_int_ioctl_func *)ioctl_enum_framesizes },
      { .num = vidioc_int_enum_frameintervals_num,
        .func = (v4l2_int_ioctl_func *)ioctl_enum_frameintervals },
      { .num = vidioc_int_dev_init_num,
        .func = (v4l2_int_ioctl_func *)ioctl_dev_init },
      { .num = vidioc_int_dev_exit_num,
        .func = (v4l2_int_ioctl_func *)ioctl_dev_exit },
      { .num = vidioc_int_s_power_num,
        .func = (v4l2_int_ioctl_func *)ioctl_s_power },
      { .num = vidioc_int_g_priv_num,
        .func = (v4l2_int_ioctl_func *)ioctl_g_priv },
      { .num = vidioc_int_g_ifparm_num,
        .func = (v4l2_int_ioctl_func *)ioctl_g_ifparm },
      { .num = vidioc_int_init_num,
        .func = (v4l2_int_ioctl_func *)ioctl_init },
      { .num = vidioc_int_enum_fmt_cap_num,
        .func = (v4l2_int_ioctl_func *)ioctl_enum_fmt_cap },
      { .num = vidioc_int_try_fmt_cap_num,
        .func = (v4l2_int_ioctl_func *)ioctl_try_fmt_cap },
      { .num = vidioc_int_g_fmt_cap_num,
        .func = (v4l2_int_ioctl_func *)ioctl_g_fmt_cap },
      { .num = vidioc_int_s_fmt_cap_num,
        .func = (v4l2_int_ioctl_func *)ioctl_s_fmt_cap },
      { .num = vidioc_int_g_parm_num,
        .func = (v4l2_int_ioctl_func *)ioctl_g_parm },
      { .num = vidioc_int_s_parm_num,
        .func = (v4l2_int_ioctl_func *)ioctl_s_parm },
      { .num = vidioc_int_queryctrl_num,
        .func = (v4l2_int_ioctl_func *)ioctl_queryctrl },
      { .num = vidioc_int_g_ctrl_num,
        .func = (v4l2_int_ioctl_func *)ioctl_g_ctrl },
      { .num = vidioc_int_s_ctrl_num,
        .func = (v4l2_int_ioctl_func *)ioctl_s_ctrl },
      { .num = vidioc_int_s_video_routing_num,
        .func = (v4l2_int_ioctl_func *)ioctl_s_routing },
    };


    static struct v4l2_int_slave fsr172x_slave = {
      .ioctls = fsr172x_ioctl_desc,
      .num_ioctls = ARRAY_SIZE(fsr172x_ioctl_desc),
    };

    static struct v4l2_int_device fsr172x_int_device = {
      .module = THIS_MODULE,
      .name = "fsr172x",
      .priv = &fsr172x,
      .type = v4l2_int_type_slave,
      .u = {
        .slave = &fsr172x_slave,
      },
    };


Note: Most of the `ioctl_x_yz` functions refer to `V4L2`s `VIDIOC_X_YZ` `#define`s.

Appendix
=======

  * `V4L2_FRMSIZE_TYPE_DISCRETE` vs `V4L2_FRMSIZE_TYPE_STEPWISE` vs `V4L2_FRMSIZE_TYPE_CONTINUOUS`
    * discrete: finite values that don't change
    * step-wise: ??
    * continuous: ranges from a `min` to a `max` by a value of `step`
