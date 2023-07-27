#### `FrameBuffer`应用编程

> Frame的意思是帧，buffer的意思是缓冲区。`Framebuffer`就是一块内存(硬件设备)，里面保存着一帧图像。

> 在 Linux 中应用程序是通过操作 RGB LCD 的显存来实现在 LCD 上显示字符、图片 等信息。在裸机中我们可以随意的分配显存，但是在 Linux 系统中内存的管理很严格，显存是需要申请的，不是想用就能用的。而且因为虚拟内存的存在，**驱动程序设置的显存和应用程序访问的显存要是同一片物理内存**。 为了解决上述问题，Framebuffer 诞生了。

##### `FrameBuffer`设备驱动

帧缓冲（framebuffer）是 Linux 系统中的一种显示驱动接口，它将显示设备（譬如 LCD） 进行抽象、 屏蔽了不同显示设备硬件的实现，对应用层抽象为一块显示内存（显存），它允许上层应用程序直接对显示缓冲区进行读写操作，而用户不必关心物理显存的位置等具体细节，这些都由Framebuffer设备驱动来完成。

所以在 Linux 系统中，显示设备被称为 FrameBuffer 设备（帧缓冲设备），所以 LCD 显示屏自然而言 就是 FrameBuffer 设备。FrameBuffer 设备对应的设备文件为`/dev/fbX`（X 为数字，0、1、2、3 等），Linux 下可支持多个 FrameBuffer 设备，最多可达 32 个，分别为`/dev/fb0` 到`/dev/fb31`，开发板出厂系统中，`/dev/fb0` 设备节点便是 LCD 屏。

##### 操作`/dev/fbX` 的一般步骤

①、首先打开`/dev/fbX` 设备文件；

②、使用 `ioctl()`函数获取到当前显示设备的参数信息，譬如屏幕的分辨率大小、像素格式，根据屏幕参 数计算显示缓冲区的大小；

 ③、通过存储映射 I/O 方式将屏幕的显示缓冲区映射到用户空间（`mmap`）；

 ④、映射成功后就可以直接读写屏幕的显示缓冲区，进行绘图或图片显示等操作了；

 ⑤、完成显示后，调用 `munmap()`取消映射、并调用 `close()`关闭设备文件。

##### 使用 `mmap()`将显示缓冲区映射到用户空间

> **存储映射 I/O** 的一个非常经典的使用场景便 是用在 `Framebuffer` 应用编程中。
>
> **通过 `mmap()`将显示器的显示缓冲区（显存）映射到进程的地址空间**中，这 样应用程序便可直接对显示缓冲区进行读写操作。

> 一块内存与LCD的像素一一对应：
>
> 1. LCD上面显示的图像色彩，由其对应的内存的数据决定
> 2. 映射内存的大小至少得等于LCD的真实尺寸大小
> 3. **映射内存的大小可以大于LCD的真实尺寸，有利于优化动态画面（视频）体验**



##### 屏幕参数设定

```c
struct fb_fix_screeninfo
{
    char id[16];              /* identification string eg "TT Builtin" */
    unsigned long smem_start; /* Start of frame buffer mem */
                              /* (physical address) */
    __u32 smem_len;           /* Length of frame buffer mem */
    __u32 type;               /* see FB_TYPE_*        */
    __u32 type_aux;           /* Interleave for interleaved Planes */
    __u32 visual;             /* see FB_VISUAL_*        */ 
    __u16 xpanstep;           /* zero if no hardware panning  */
    __u16 ypanstep;           /* zero if no hardware panning  */
    __u16 ywrapstep;          /* zero if no hardware ywrap    */
    __u32 line_length;        /* length of a line in bytes    */
    ...
    ...
};

struct fb_var_screeninfo
{
    __u32 xres;           /* 可见区宽度（单位：像素） */
    __u32 yres;           /* 可见区高度（单位：像素） */
    __u32 xres_virtual;   /* 虚拟区宽度（单位：像素） */
    __u32 yres_virtual;   /* 虚拟区高度（单位：像素） */
    __u32 xoffset;        /* 虚拟区到可见区x轴偏移量 */
    __u32 yoffset;        /* 虚拟区到可见区y轴偏移量 */

    __u32 bits_per_pixel; /* 色深 */

    // 像素内颜色结构
    struct fb_bitfield red;   // 红色  
    struct fb_bitfield green; // 绿色
    struct fb_bitfield blue;  // 蓝色
    struct fb_bitfield transp;// 透明度
    ...
    ...
};
```

上述结构体的具体定义在系统的如下路径中：

```bash
/usr/include/linux/fb.h
```

```c
/*
vinfo.xres 是帧缓冲的水平分辨率，即屏幕的宽度（以像素为单位）。
vinfo.yres 是帧缓冲的垂直分辨率，即屏幕的高度（以像素为单位）。
vinfo.bits_per_pixel 是每个像素所占的位数。它表示每个像素使用多少位来表示颜色信息。
*/
	struct fb_var_screeninfo vinfo; // 显卡设备的可变属性结构体
	ioctl(lcd, FBIOGET_VSCREENINFO, &vinfo); // 获取可变属性

    // 获得当前显卡所支持的虚拟区显存大小
    unsigned long VWIDTH  = vinfo.xres_virtual;
    unsigned long VHEIGHT = vinfo.yres_virtual;
    unsigned long BPP = vinfo.bits_per_pixel;
```

可见区、虚拟区都是内存区域，可见区是虚拟区的一部分，因此可见区尺寸至少等于虚拟区。
一般而言，可见区尺寸就是屏幕尺寸，比如320×240；而虚拟区是显示设备能支持的显存大小，比如320×240、800×480、800×960等。
为了提高画面体验，一般先在不可见区操作显存数据，然后在调整可见区位置，使得图像“瞬间”呈现，避免闪屏。

> 成员变量xres 和 yres定义在显示屏上真实显示的分辨率。而xres_virtual和yres_virtual是虚拟分辨率，它们定义的是**显存分辨率**。

![](imgs\帧缓冲设备.png)
