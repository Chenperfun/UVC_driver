# 前言

​		通常来说，产品的驱动程序不需要编写，Linux内核提供了足够完善的驱动供我们使用。但是，本着追求进步、实事求是、天天向上的理念（xian de dan teng），最终完成了本鸿篇巨作之USB摄像头驱动（UVC），正所谓：学好数理化以及UVC，走遍天下全不怕。瞎扯到此为止，接下来开始认真的记录uvc驱动学习过程。驱动不完善，还有很多不准确的地方，望各位不吝赐教 :D，最后，提前感谢各位的阅读。



# 一、uvc驱动简述

​		UVC全称为USB Video Class，即：USB视频类，是一种为USB视频捕获设备定义的协议标准。是Microsoft与另外几家设备厂商联合推出的为USB视频捕获设备定义的协议标准，已成为USB org标准之一。

​		如今的主流操作系统(如Windows XP SP2 and later, Linux 2.4.6 and later, MacOS 10.5 and later)都已提供UVC设备驱动，因此符合UVC规格的硬件设备在不需要安装任何的驱动程序下即可在主机中正常使用。使用UVC技术的包括摄像头、数码相机、类比影像转换器、电视棒及静态影像相机等设备。

​		最新的UVC版本为UVC 1.5，由USB Implementers Forum定义包括基本协议及负载格式。

​		网络摄像头是第一个支持UVC而且也是数量最多的UVC设备，操作系统只要是 Windows XP SP2 之后的版本都可以支持 UVC，当然 Vista 就更不用说了。Linux系统自2.4以后的内核都支持了大量的设备驱动，并可以支持UVC设备。



# 二、uvc驱动框架搭建

## 2.1 理理思路		

​		UVC驱动首先是属于USB驱动下的一种，USB驱动向上能发现更大的驱动类：字符设备驱动。所以，考虑首先应该从字符设备驱动入手。



## 2.2 字符设备驱动分析

![](C:\Users\Administrator\Desktop\摄像头驱动——UVC驱动分析及编写\字符设备驱动.png)



​		根据linux字符设备驱动的常见套路，APP在应用层调用read、write、open等接口，调用库函数，触发swi软件异常，进入内核，最终会调用到驱动程序的open、read、write等等。



## 2.3 UVC入口函数

​		接下来正式进入到UVC驱动的分析，首先，需要一个入口函数以及出口函数。由于UVC属于USB设备，所以在入口函数中注册usb_driver：

```c
static int myuvc_init(void)
{
    usb_register(&myuvc_driver);
    return 0;
}

static void myuvc_exit(void)
{
    usb_deregister(&myuvc_driver);
}

module_init(myuvc_init);
module_exit(myuvc_exit);
MODULE_LICENSE("GPL");
```

​		这里的usb_driver应当包含如下成员：

```c
static struct usb_driver myuvc_driver = {
    .name       = "myuvc",
    .probe      = myuvc_probe,
    .disconnect = myuvc_disconnect,
    .id_table   = myuvc_ids,
};
```

​		当在系统中insmod装载该驱动程序时，会在入口函数直接注册usb_drvier结构体，通过比较设备提供的INFO，和id_table比较，若匹配，则表明驱动支持该usb，uvc驱动的id_table可由USB的接口自动生成，如下所示：

```c
static struct usb_device_id myuvc_ids[] = {
	/* Generic USB Video Class */
	{ USB_INTERFACE_INFO(USB_CLASS_VIDEO, 1, 0) },  /* VideoControl Interface */
    { USB_INTERFACE_INFO(USB_CLASS_VIDEO, 2, 0) },  /* VideoStreaming Interface */
	{}
};
```

​		这里介绍一下，一个uvc摄像头设备被Linux分为2个逻辑接口，分别为控制接口：**VideoControl Interface**；以及传输接口：**VideoStreaming Interface**。USB通过设备描述符的形式，控制和管理设备。在Linux系统中，可以通过lsusb查看某一级的描述符，比如 lsusb -v -d 0x1e4e: "bBitsPerPixel"，可以查看传输数据的格式：字节每像素。

​		最后，相应的，在出口函数中应当释放注册的usb_driver驱动。



## 2.4 uvc设备

​		在驱动入口函数中，比较id_table，若匹配，则调用usb_driver的probe函数。probe函数如下所示：

```c
static int myuvc_probe(struct usb_interface *intf,
		     const struct usb_device_id *id)
{
    static int cnt = 0;
	struct usb_device *dev = interface_to_usbdev(intf);

    myuvc_udev = dev;

    if (cnt == 1)
    {
        myuvc_control_intf = intf->cur_altsetting->desc.bInterfaceNumber;
    }
    else if (cnt == 2)
    {
        myuvc_streaming_intf = intf->cur_altsetting->desc.bInterfaceNumber;
    }

    if (cnt == 2)
    {
        /* 1. 分配一个video_device结构体 */
        myuvc_vdev = video_device_alloc();

        /* 2. 设置 */
        myuvc_vdev->release = myuvc_release;
        
        myuvc_vdev->fops    = &myuvc_fops;
        
        myuvc_vdev->ioctl_ops = &myuvc_ioctl_ops;

        /* 3. 注册 */
        video_register_device(myuvc_vdev, VFL_TYPE_GRABBER, -1);
    }
    return 0;
}
```

​		上文提到过，uvc有两个逻辑接口，一个用于传输，一个用于控制。所以，正常情况下，一个uvc设备应当被注册了两次。在这里，通过动态注册的方法，设置usb_device。由于在内核中，要求必须有usb_release函数，所以添加，但是为一个空函数。最主要的是**fops**以及**ioctl_ops**。

```c
static const struct v4l2_file_operations myuvc_fops = {
	.owner		= THIS_MODULE,
    .open       = myuvc_open,
    .release    = myuvc_close,
    .mmap       = myuvc_mmap,
    .ioctl      = video_ioctl2, 	/* V4L2 ioctl handler */
    .poll       = myuvc_poll,		
};
```

​		APP在应用空间调用open函数，最终调用到uvc_open，打开uvc摄像头设备，通过mmap映射虚拟地址，用于存放摄像头传输的数据队列，接下来重点分析一下设备的ioctl_ops。



### 2.4.1 uvc设备的ioctl_ops

​		uvc设备的ioctl_ops由14个重要操作函数组成，他们的作用如下注释：

```c
static const struct v4l2_ioctl_ops myuvc_ioctl_ops = {
        // 表示它是一个摄像头设备
        .vidioc_querycap      = myuvc_vidioc_querycap,

        /* 用于列举、获得、测试、设置摄像头的数据的格式 */
        .vidioc_enum_fmt_vid_cap  = myuvc_vidioc_enum_fmt_vid_cap,
        .vidioc_g_fmt_vid_cap     = myuvc_vidioc_g_fmt_vid_cap,
        .vidioc_try_fmt_vid_cap   = myuvc_vidioc_try_fmt_vid_cap,
        .vidioc_s_fmt_vid_cap     = myuvc_vidioc_s_fmt_vid_cap,
        
        /* 缓冲区操作: 申请/查询/放入队列/取出队列 */
        .vidioc_reqbufs       = myuvc_vidioc_reqbufs,
        .vidioc_querybuf      = myuvc_vidioc_querybuf,
        .vidioc_qbuf          = myuvc_vidioc_qbuf,
        .vidioc_dqbuf         = myuvc_vidioc_dqbuf,

        /* 查询/获得/设置属性 */
        .vidioc_queryctrl     = myuvc_vidioc_queryctrl,
        .vidioc_g_ctrl        = myuvc_vidioc_g_ctrl,
        .vidioc_s_ctrl        = myuvc_vidioc_s_ctrl,
        
        // 启动/停止
        .vidioc_streamon      = myuvc_vidioc_streamon,
        .vidioc_streamoff     = myuvc_vidioc_streamoff,   
};
```

​		从这里可以看出uvc设备的ioctl_ops涵盖了整个开始到传输再到结束的流程，ioctl_opsuvc是重点分析对象，接下来通过分析该控制结构体，来揭示uvc摄像头驱动的神秘面纱~



# 三、传输流程

​		讲解的流程分为控制接口：**VideoControl Interface**；以及传输接口：**VideoStreaming Interface**。通过传输接口完成：底层数据的搜集，上报，传输等过程。通过控制接口完成：音量控制，亮度调节等具体参数控制。



## 3.1 传输流程

​		当APP最终调用到驱动程序open。

​		首先是摄像头格式的设置操作：

1. myuvc_vidioc_querycap，表明设备为视频捕捉设备。

2. myuvc_vidioc_enum_fmt_vid_cap，列举驱动支持哪些格式。简单设置为支持一种格式YUYV。

3. myuvc_vidioc_g_fmt_vid_cap，返回当前所使用的格式。

4. myuvc_vidioc_try_fmt_vid_cap，测试驱动程序是否支持某种格式, 强制设置该格式。

5. myuvc_vidioc_s_fmt_vid_cap，确认第四步尝试的格式，并进一步设置细节。

   

   接下来是为传输准备空间和队列：

6. myuvc_vidioc_reqbufs，若干个缓存, APP将从这些缓存中读到视频数据。

7. myuvc_vidioc_querybuf，查询缓存状态, 比如地址信息(APP可以用mmap进行映射) 。

8. myuvc_mmap，此函数在device_fops中，将分配好的缓存映射到用户空间，供APP直接操作。

9. myuvc_vidioc_qbuf，把缓冲区放入队列, 底层的硬件操作函数将会把数据放入这个队列的缓存。

   ​	

   万事俱备，开始传输：

10. myuvc_vidioc_streamon，启动传输。

11. myuvc_poll，APP调用POLL/select来确定缓存是否就绪(有数据) 。

12. myuvc_vidioc_dqbuf，若有数据，则把缓存从队列取出，mmap映射过内存，所以APP可直接操作缓存。

13. 重复查询、获取......

14. myuvc_vidioc_streamoff，传输结束，停止。

​		

## 3.2 控制流程

​		控制相对简单：

1. myuvc_vidioc_queryctrl，查询想要设置的属性，在设备中是否可用。
2. myuvc_vidioc_g_ctrl，从设备获取属性。
3. myuvc_vidioc_s_ctrl，向设备设置属性。



# 四、正式编写

​		在清楚了整个uvc的传输以及控制流程之后，开始在框架下开始编写uvc驱动。编写按照框架所罗列的步骤，依次填充框架，最后将完善为一个可用的uvc驱动。

​		不管是vs接口（传输），还是vc接口（控制），都是从设备的open函数作为入口，暂时不需要做什么工作，所以讲open函数暂定为空。

​		

## 4.1 数据传输

### 4.1.1 格式设置

1. myuvc_vidioc_querycap，表明设备为视频捕捉设备。

   ```c
   static int myuvc_vidioc_querycap(struct file *file, void  *priv,
   					struct v4l2_capability *cap)
   {    
       memset(cap, 0, sizeof *cap);
       strcpy(cap->driver, "myuvc");
       strcpy(cap->card, "myuvc");
       cap->version = 1;
       
       cap->capabilities = V4L2_CAP_VIDEO_CAPTURE | V4L2_CAP_STREAMING;
    
   	return 0;
   }
   ```

   函数的任务就是确定设备是一个视频捕捉设备，且传输方式是视频流（区别于读写RW）。

   

2. myuvc_vidioc_enum_fmt_vid_cap，列举驱动支持哪些格式。简单设置为支持一种格式YUYV。

   为方便前期开发，只设置一种支持格式，后续格式可后期添加。

   ```c
   static int myuvc_vidioc_enum_fmt_vid_cap(struct file *file, void  *priv,
   					struct v4l2_fmtdesc *f)
   {
       /* 人工查看描述符可知我们用的摄像头只支持1种格式 */
   	if (f->index >= 1)
   		return -EINVAL;
   
      
   	strcpy(f->description, "4:2:2, packed, YUYV");
   	f->pixelformat = V4L2_PIX_FMT_YUYV;    
       
   	return 0;
   }
   ```

   ​		 支持什么格式呢？查看VideoStreaming Interface的描述符，得到GUID为"59 55 59 32 00 00 10 00 80 00 00 aa 00 38 9b 71"，也就是YUYV。

   

3. myuvc_vidioc_g_fmt_vid_cap，返回当前所使用的格式。

   ```c
   static int myuvc_vidioc_g_fmt_vid_cap(struct file *file, void *priv,
   					struct v4l2_format *f)
   {
       memcpy(f, &myuvc_format, sizeof(myuvc_format));
   	return (0);
   }
   ```

   

4. myuvc_vidioc_try_fmt_vid_cap，测试驱动程序是否支持某种格式, 强制设置该格式，并设置该格式下的属性参数。

   ```c
   static int myuvc_vidioc_try_fmt_vid_cap(struct file *file, void *priv,
   			struct v4l2_format *f)
   {
       if (f->type != V4L2_BUF_TYPE_VIDEO_CAPTURE)
       {
           return -EINVAL;
       }
       if (f->fmt.pix.pixelformat != V4L2_PIX_FMT_YUYV)
           return -EINVAL; 
       /* 调整format的width, height, 
        * 计算bytesperline, sizeimage
        */
   
       /* 人工查看描述符, 确定支持哪几种分辨率 */
       f->fmt.pix.width  = frames[frame_idx].width;
       f->fmt.pix.height = frames[frame_idx].height;
       
   	f->fmt.pix.bytesperline =
   		(f->fmt.pix.width * bBitsPerPixel) >> 3;
   	f->fmt.pix.sizeimage =
   		f->fmt.pix.height * f->fmt.pix.bytesperline;
       
       return 0;
   }
   ```

   这里只设置了分辨率。

   

5. myuvc_vidioc_s_fmt_vid_cap，确认第四步尝试的格式，并进一步设置细节。

```c
static int myuvc_vidioc_s_fmt_vid_cap(struct file *file, void *priv,
					struct v4l2_format *f)
{
	int ret = myuvc_vidioc_try_fmt_vid_cap(file, NULL, f);
	if (ret < 0)
		return ret;

    memcpy(&myuvc_format, f, sizeof(myuvc_format));
    
    return 0;
}
```

​		将上一步尝试的格式和参数，设置项SUB摄像头。



### 4.1.2 为传输准备空间和队列

1. myuvc_vidioc_reqbufs，若干个缓存, APP将从这些缓存中读到视频数据。

   函数部分重要代码片段如下所示：

   ```c
   int nbuffers = p->count;		//分配缓冲区个数
   int bufsize  = PAGE_ALIGN(myuvc_format.fmt.pix.sizeimage);	//每个缓冲区的大小按页对齐
   ... ...
           /* 这些缓存是一次性作为一个整体来分配的 */
       memset(&myuvc_queue, 0, sizeof(myuvc_queue));
   
   	INIT_LIST_HEAD(&myuvc_queue.mainqueue);
   	INIT_LIST_HEAD(&myuvc_queue.irqqueue);
   
       for (i = 0; i < nbuffers; ++i) {
           myuvc_queue.buffer[i].buf.index = i;
           myuvc_queue.buffer[i].buf.m.offset = i * bufsize;
           myuvc_queue.buffer[i].buf.length = myuvc_format.fmt.pix.sizeimage;
           myuvc_queue.buffer[i].buf.type = V4L2_BUF_TYPE_VIDEO_CAPTURE;
           myuvc_queue.buffer[i].buf.sequence = 0;
           myuvc_queue.buffer[i].buf.field = V4L2_FIELD_NONE;
           myuvc_queue.buffer[i].buf.memory = V4L2_MEMORY_MMAP;
           myuvc_queue.buffer[i].buf.flags = 0;
           myuvc_queue.buffer[i].state     = VIDEOBUF_IDLE;	//设置为空闲状态
           init_waitqueue_head(&myuvc_queue.buffer[i].wait);	//队列等待
       }
   
       myuvc_queue.mem = mem;
       myuvc_queue.count = nbuffers;
       myuvc_queue.buf_size = bufsize;
       ret = nbuffers;
   ```

   这里用到一个关键的编程技巧，两个队列头管理一个缓冲队列。当有数据时，从irqqueue取出第一个buf1，放入数据。当应用程序通过poll查询到有数据时，从mainqueue取出第一个buf1，取出数据，初始化buf1，并入列队尾。具体的队列操作请见板块[五、问题与解决](#queue1);

   ​     

2. myuvc_vidioc_querybuf，查询缓存状态, 比如地址信息(APP可以用mmap进行映射) 。

   部分代码片段如下：

   ```c
       memcpy(v4l2_buf, &myuvc_queue.buffer[v4l2_buf->index].buf, sizeof(*v4l2_buf));
   
   	if (myuvc_queue.buffer[v4l2_buf->index].vma_use_count)
   		v4l2_buf->flags |= V4L2_BUF_FLAG_MAPPED;
   
   	switch (myuvc_queue.buffer[v4l2_buf->index].state) {
       	case VIDEOBUF_ERROR:
       	case VIDEOBUF_DONE:
       		v4l2_buf->flags |= V4L2_BUF_FLAG_DONE;
       		break;
       	case VIDEOBUF_QUEUED:
       	case VIDEOBUF_ACTIVE:
       		v4l2_buf->flags |= V4L2_BUF_FLAG_QUEUED;
       		break;
       	case VIDEOBUF_IDLE:
       	default:
       		break;
   ```

   可以看到，函数直接根据队列中缓存的索引（数组下标），取出缓存，复制到v4l2_buf。除此以外，函数还更新了缓存的状态。

   

3. myuvc_mmap，此函数在device_fops中，将分配好的缓存映射到用户空间，供APP直接操作。

   函数调用到了vma，部分代码：

   ```c
       for (i = 0; i < myuvc_queue.count; ++i) {
           buffer = &myuvc_queue.buffer[i];
           if ((buffer->buf.m.offset >> PAGE_SHIFT) == vma->vm_pgoff)
               break;
       }
   	... ...
       addr = (unsigned long)myuvc_queue.mem + buffer->buf.m.offset;
       while (size > 0) {
           page = vmalloc_to_page((void *)addr);
   
           /* 把page和APP传入的虚拟地址挂构 */
           if ((ret = vm_insert_page(vma, start, page)) < 0)
               goto done;
   
           start += PAGE_SIZE;
           addr += PAGE_SIZE;
           size -= PAGE_SIZE;
   ```

   首先取出每一个缓存，分别对每一个缓存与虚拟地址，按页挂钩。

   

4. myuvc_vidioc_qbuf，把缓冲区放入队列, 底层的硬件操作函数将会把数据放入这个队列的缓存。

   ```c
       /* 1. 修改状态 */
   	buf->state = VIDEOBUF_QUEUED;
   	buf->buf.bytesused = 0;
   
       /* 2. 放入2个队列 */
       /* 队列1: 供APP使用 
        * 当缓冲区没有数据时,放入mainqueue队列
        * 当缓冲区有数据时, APP从mainqueue队列中取出
        */
   	list_add_tail(&buf->stream, &myuvc_queue.mainqueue);
   
       /* 队列2: 供产生数据的函数使用
        * 当采集到数据时,从irqqueue队列中取出第1个缓冲区,存入数据
        */
   	list_add_tail(&buf->irq, &myuvc_queue.irqqueue);
       
   	return 0;
   ```

   可以发现，这里用到了之前提到的两个对队列头，详情请见5.1节。



### 4.1.3 启动传输

myuvc_vidioc_streamon，启动传输。

​		在完成了属性设置，以及构造数据队列以后，即将开始UVC最复杂的一部分：数据传输。数据传输可以分为两个部分：通过msg向硬件传递参数；分配构造urb包。

​		传输用到的是VideoStream接口，需要单独设置启动的参数，在这里，启动参数用msg包进行传输，VS接口所需的参数如下结构体：

```c
struct myuvc_streaming_control {
	__u16 bmHint;
	__u8  bFormatIndex;
	__u8  bFrameIndex;
	__u32 dwFrameInterval;
	__u16 wKeyFrameRate;
	__u16 wPFrameRate;
	__u16 wCompQuality;
	__u16 wCompWindowSize;
	__u16 wDelay;
	__u32 dwMaxVideoFrameSize;
	__u32 dwMaxPayloadTransferSize;
	__u32 dwClockFrequency;
	__u8  bmFramingInfo;
	__u8  bPreferedVersion;
	__u8  bMinVersion;
	__u8  bMaxVersion;
};
```

可以看到，这里用到前边设置好的属性和参数，这个结构体的数据，可以通过手动逐条设置，但是内核提供了独特的方法，可直接通过USB的描述符中解析出来。

```c
static int myuvc_vidioc_streamon(struct file *file, void *priv, enum v4l2_buf_type i)
{
    int ret;
    /* a. 测试参数 */
    ret = myuvc_try_streaming_params(&myuvc_params);
    printk("myuvc_try_streaming_params ret = %d\n", ret);

    /* b. 取出参数 */
    ret = myuvc_get_streaming_params(&myuvc_params);
    printk("myuvc_get_streaming_params ret = %d\n", ret);

    /* c. 设置参数 */
    ret = myuvc_set_streaming_params(&myuvc_params);
    printk("myuvc_set_streaming_params ret = %d\n", ret);
```

在streamon中，调用三个封装好的函数，用类似于上文参数获取的方法，测试、取出并设置参数。具体的设置过程不多赘述，下面主要来看一看参数是如何传递给硬件的。

```c
static int myuvc_try_streaming_params(struct myuvc_streaming_control *ctrl)
{
	... ...
    pipe = (SET_CUR & 0x80) ? usb_rcvctrlpipe(myuvc_udev, 0)
                  : usb_sndctrlpipe(myuvc_udev, 0);
    type |= (SET_CUR & 0x80) ? USB_DIR_IN : USB_DIR_OUT;

    ret = usb_control_msg(myuvc_udev, pipe, SET_CUR, type, VS_PROBE_CONTROL << 8,
            0 << 8 | myuvc_streaming_intf, data, size, 5000);
	... ...
}
```

上图调用usb_control_msg函数，将设置好的msg包传递到硬件。下面来看看try、set、get的区别：

```c
static int myuvc_get_streaming_params(struct myuvc_streaming_control *ctrl)
{
	... ...
	pipe = (GET_CUR & 0x80) ? usb_rcvctrlpipe(myuvc_udev, 0)
			      : usb_sndctrlpipe(myuvc_udev, 0);
	type |= (GET_CUR & 0x80) ? USB_DIR_IN : USB_DIR_OUT;

	ret = usb_control_msg(myuvc_udev, pipe, GET_CUR, type, VS_PROBE_CONTROL << 8,
			0 << 8 | myuvc_streaming_intf, data, size, 5000);
	... ...
}
```

​		get和try的区别在于：分别为SET_CUR与GET_CUR标志，二者都是VS_PROBE_CONTROL，代表只是连接过程，并非真正的设置过程。

```c
static int myuvc_set_streaming_params(struct myuvc_streaming_control *ctrl)
{
    ... ...
    pipe = (SET_CUR & 0x80) ? usb_rcvctrlpipe(myuvc_udev, 0)
                  : usb_sndctrlpipe(myuvc_udev, 0);
    type |= (SET_CUR & 0x80) ? USB_DIR_IN : USB_DIR_OUT;

    ret = usb_control_msg(myuvc_udev, pipe, SET_CUR, type, VS_COMMIT_CONTROL << 8,
            0 << 8 | myuvc_streaming_intf, data, size, 5000);

	... ...
}
```

​		try和set区别在于：二者标志都是SET_CUR，但分别为VS_PROBE_CONTROL和VS_COMMIT_CONTROL，代表try是连接过程，set是最终确认过程。



​		urb包类似于网络socket编程中的sk_buf包，用于usb协议中硬件上报传输数据。在streamon函数中，分配

并设置，在发包前，还需要一个接口设置，usb_set_interface。来设置带宽：根据setting的endpoint能传输的wMaxPacketSize，找到能满足该带宽的setting。

```c
    usb_set_interface(myuvc_udev, myuvc_streaming_intf,myuvc_streaming_bAlternateSetting);
    
	ret = myuvc_alloc_init_urbs();
    if (ret)
        printk("myuvc_alloc_init_urbs err : ret = %d\n", ret);
```

​		最后，提交URB包，准备开始接受数据：

```c
	for (i = 0; i < MYUVC_URBS; ++i) {
		if ((ret = usb_submit_urb(myuvc_queue.urb[i], GFP_KERNEL)) < 0) {
			printk("Failed to submit URB %u (%d).\n", i, ret);
			myuvc_uninit_urbs();
			return ret;
		}
	}
    
	return 0;
}
```

​		至此，STREAMON启动全部完成。



### 4.1.4 查询获取数据

​		接下来就可以开始从硬件接受数据了。

1. myuvc_poll，APP调用POLL/select来确定缓存是否就绪(有数据) 。

   从mainqueue队列头中，取出第一个buf1，若未就绪，则休眠等待。

   ```c
       buf = list_first_entry(&myuvc_queue.mainqueue, struct myuvc_buffer, stream);
   
       poll_wait(file, &buf->wait, wait);
   ```

   

2. myuvc_vidioc_dqbuf，若有数据，则把缓存从队列取出，mmap映射过内存，所以APP可直接操作缓存。

   ```c
   	buf = list_first_entry(&myuvc_queue.mainqueue, struct myuvc_buffer, stream);
   
   	switch (buf->state) {
   	case VIDEOBUF_ERROR:
   		ret = -EIO;
   	case VIDEOBUF_DONE:
   		buf->state = VIDEOBUF_IDLE;
   		break;
   
   	case VIDEOBUF_IDLE:
   	case VIDEOBUF_QUEUED:
   	case VIDEOBUF_ACTIVE:
   	default:
   		ret = -EINVAL;
   		goto done;
   	}
   
   	list_del(&buf->stream);
   ```

   若有数据，则取出buf1，处理完数据后，入列到队列mainqueue尾部。并更新了缓冲区状态。

3. 重复查询、获取......

4. myuvc_vidioc_streamoff，传输结束，停止。

```c
static int myuvc_vidioc_streamoff(struct file *file, void *priv, enum v4l2_buf_type t)
{
	struct urb *urb;
	unsigned int i;

    /* 1. kill URB */
	for (i = 0; i < MYUVC_URBS; ++i) {
		if ((urb = myuvc_queue.urb[i]) == NULL)
			continue;
		usb_kill_urb(urb);
	}

    /* 2. free URB */
    myuvc_uninit_urbs();

    /* 3. 设置VideoStreaming Interface为setting 0 */
    usb_set_interface(myuvc_udev, myuvc_streaming_intf, 0);
    
    return 0;
}
```

​		调用streamoff关闭传输，主要做了三部分工作：

​		清空urb包，释放urb缓冲内存，将VS传输接口关闭。



以上就是UVC驱动传输的全过程。可以看出，UVC驱动比较复杂，难点集中在启动传输过程中。



## 4.2 属性设置

接下来是比较轻松的属性设置环节，分为三个步骤：

​    /* 查询/获得/设置属性 */

​    .vidioc_queryctrl   = myuvc_vidioc_queryctrl,

​    .vidioc_g_ctrl    = myuvc_vidioc_g_ctrl,

​    .vidioc_s_ctrl    = myuvc_vidioc_s_ctrl,

依次分析。

1. myuvc_vidioc_queryctrl，查询USB设备支持设置的属性值。属性设置属于VideoControl接口，同样，可以根据USB的描述符，定位到可以设置的属性，查询的到设备可以设置亮度：

```c
	memset(ctrl, 0, sizeof *ctrl);
	ctrl->id   = V4L2_CID_BRIGHTNESS;
	ctrl->type = V4L2_CTRL_TYPE_INTEGER;
	strcpy(ctrl->name, "MyUVC_BRIGHTNESS");
	ctrl->flags = 0;
```

查询到亮度可调节，但是调节的细节还需要确认，确认过程同样是依赖MSG包：

查询亮度调节的最小值：

```c
	ret = usb_control_msg(myuvc_udev, pipe, GET_MIN, type, PU_BRIGHTNESS_CONTROL << 8,
			ProcessingUnitID << 8 | myuvc_control_intf, data, 2, 5000);
    if (ret != 2)
        return -EIO;
	ctrl->minimum = myuvc_get_le_value(data);	/* Note signedness */

```

查询亮度调节最大值：

```c
	ret = usb_control_msg(myuvc_udev, pipe, GET_MAX, type,  PU_BRIGHTNESS_CONTROL << 8,
			ProcessingUnitID << 8 | myuvc_control_intf, data, 2, 5000);
    if (ret != 2)
        return -EIO;
	ctrl->maximum = myuvc_get_le_value(data);	/* Note signedness */
```

查询亮度调节的步阶：

```c
	ret = usb_control_msg(myuvc_udev, pipe, GET_RES, type, PU_BRIGHTNESS_CONTROL << 8,
			 ProcessingUnitID << 8 | myuvc_control_intf, data, 2, 5000);
    if (ret != 2)
        return -EIO;
	
	ctrl->step = myuvc_get_le_value(data);	/* Note signedness */
```

查询亮度调节的默认值：

```c
ret = usb_control_msg(myuvc_udev, pipe, GET_DEF, type, PU_BRIGHTNESS_CONTROL << 8,
		ProcessingUnitID << 8 | myuvc_control_intf, data, 2, 5000);
if (ret != 2)
    return -EIO;
ctrl->default_value = myuvc_get_le_value(data);	/* Note signedness */
```
​		

2. myuvc_vidioc_g_ctrl，用于获取属性。

   ```c
   	ret = usb_control_msg(myuvc_udev, pipe, GET_CUR, type, PU_BRIGHTNESS_CONTROL << 8,
   			ProcessingUnitID << 8 | myuvc_control_intf, data, 2, 5000);
       if (ret != 2)
           return -EIO;
   	ctrl->value = myuvc_get_le_value(data);	/* Note signedness */
   ```



3. myuvc_vidioc_s_ctrl，最终的属性设置。

```c
    ret = usb_control_msg(myuvc_udev, pipe, SET_CUR, type, PU_BRIGHTNESS_CONTROL << 8,
            ProcessingUnitID  << 8 | myuvc_control_intf, data, 2, 5000);
```

以上以亮度调节作为例子，分析学习了属性调节的过程，到此，一个UVC驱动正式完成。



# 五、问题与解决



## <a id="queue1">5.1 两个队列头控制同一个队列</a>

​		在驱动启动传输后，用两个队列头来管理缓存队列：mainqueue，irqqueue。当底层有数据时，从irqueue队列取出第一个缓存，将数据放入缓存中。

![](C:\Users\Administrator\Desktop\摄像头驱动——UVC驱动分析及编写\队列1.png)

​		当硬件产生数据，从irqqueue取出第一个buf1，在buf1中填充数据：

![](C:\Users\Administrator\Desktop\摄像头驱动——UVC驱动分析及编写\irq队列.png)

​		应用程序通过poll调用，查询到mainqueue中有数据，则从mainqueue队列取出第一个buf1，取出buf1中的数据，并清空buf1：

![](C:\Users\Administrator\Desktop\摄像头驱动——UVC驱动分析及编写\main队列.png)

​		最后，被清空的buf1会被入列到队尾，等待下一次的数据传输：

![](C:\Users\Administrator\Desktop\摄像头驱动——UVC驱动分析及编写\读取结束队列.png)

​		至此，一次数据读取过程结束。



## 5.2 urb包的构造

​		urb包类似于网络socket编程中的sk_buf包，用于usb协议中硬件上报传输数据。注意，URB只是硬件到驱动层的数据传递，驱动通过poll查询到数据存在后，会将URB包解析，并将数据放入irqqueue队列。

​		









































