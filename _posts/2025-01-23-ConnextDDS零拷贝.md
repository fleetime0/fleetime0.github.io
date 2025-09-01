---
title: ConnextDDS零拷贝
date: 2025-01-23 8:11 +0800
last_modified_at: 2025-01-23 8:11 +0800
author: fleetime0
categories: ["DDS", "rtidds"]
tags: ["c++", "rtidds", "dds"]
pin: true
math: true
mermaid: true
img_path: /assets/img/ConnextDDS零拷贝/
---

应用场景：传输大数据时，为了减少延时，RTI Connext DDS提供了两种方式，Zero Copy transfer over shared memory和FlatData Language binding。

## 延时的产生

当使用Connext DDS时，在任何一个通用的连接框架中，延时的产生一般由三部分：middleware，copy，transport。

- **Middleware**：为了给应用程序提供对应的功能而引入中间件的操作延时，比如持久化和实例管理等。
- **Copy**：复制sample内容而引入的延时，包括序列化和反序列化。
- **Transport**：由底层传输和网络基础设施引入的延时。

![image1](image1.png)

对于数据小的sample，Copy引发的延时很小，可以忽略不计，但当数据大的时候，copy引发的延时会变大，无法满足对延时的要求。

![image2](image2.png)

![image3](image3.png)

RTI DDS使用UDP或SHMEM进行传输时，需要对sample进行四次拷贝：

1. **Copy1**：Connext DDS调用TypePlugin::serialize将sample的内存表示转换为线表示，用于存储和传输。完成序列化后，通过一个或多个可用的传输将其发送到订阅app。当底层传输的最大消息的大小比sample小时，需要进行分片。分片过程不会产生额外的拷贝。

2. **Copy2**：对于SHMEM，需要将发布过程的本地内存空间中的sample分片复制到共享内存中，订阅app将从其中读取。对于UDP，调用套接字接收的操作将复制这些分片。

3. **Copy3**：收到sample分片后，它们被重组到一个buffer中。

4. **Copy4**：Connext DDS调用TypePlugin::deserialize将sample的线表示转换为内存表示形式。

使用FlatData Language Binding时的Copy次数：

![image4](image4.png)

FlatData是一种语言绑定，其中sample的内存表示与线表示相匹配。因此，序列化和反序列化的开销为0。可以直接访问序列化的数据，而不需要提前反序列化。当使用FlatData语言绑定时，可以把之前的Copy1和Copy4取消。

使用Zero Copy transfer over shared memory或者FlatData Language binding还是两者结合，取决于DataReaders和DataWriters是否在同一台主机上，以及数据类型。下表进行了总结：

| 情况             | Readers和Writers在同一主机上 | Readers和Writers在不同主机上 | 某些在同一主机，某些在不同主机 |
| ---------------- | ---------------------------- | ---------------------------- | ------------------------------ |
| **固定大小类型** | Zero Copy                    | FlatData                     | Zero Copy和FlatData            |
| **可变大小类型** | Zero Copy和FlatData          | FlatData                     | Zero Copy和FlatData            |

## FlatData Language Bingding

创建FlatData sample时，sample buffer的内存表示为XCDR2，使用创建sample的主机的字节序来填充缓冲区。使用主机平台字节序可以快速访问sample的内容，因为设定者和获得者不需要更改字节序。

在IDL中创建数据类型时，选择FlatData进行语言绑定。示例如下：

Language_binding支持两个值：FLAT_DATA和PLAIN（default）。PLAIN指的是常规的内存表示。

对于final类型，FlatData只能被应用在固定大小类型；对于mutable类型，FlatData支持可变大小包含有界序列、有界字符串（不支持无界序列和无界字符串）。

```
enum Format {
RGB,
HSV,
YUV
};

@final
@language_binding(FLAT_DATA)
struct Resolution {
long height;
long width;
};

@final
@language_binding(FLAT_DATA)
struct Pixel {
octet red;
octet green;
octet blue;
};

const long MAX_IMAGE_SIZE = 8294400;

@mutable
@language_binding(FLAT_DATA)
struct CameraImage {
string<128> source;
Format format;
Resolution resolution;
sequence<Pixel, MAX_IMAGE_SIZE> pixels;
};
```

当一个类型被标记为FlatData语言绑定时，这种类型的样本的内存表示形式等于线表示形式，数据样本始终处于序列化格式。为了方便访问和设置样本内容，RTI Code Generator生成了辅助类型，提供创建和访问这些数据样本的操作。这些辅助类型是Samples、Offsets、Builders。

一个FlatData Sample是存储数据的线表示的缓冲区。在上面IDL生成的代码中，有一个CameraImage类型的Sample包含了这个缓冲区，它是一个可以被读写的顶层对象：

```c++
typedef rti::flat::Sample<CameraImageOffset> CameraImage;
```

要访问这个sample，应用程序使用Offset类型。Offset表示成员的类型以及它在缓冲区中的位置，可以将其看作为一个迭代器，一个指向数据但不拥有它的轻量级对象。

```c++
class NDDSUSERDllExport CameraImageConstOffset : public rti::flat::MutableOffset {
public:
    const rti::flat::StringOffset source() const;
    Format format() const;
    Resolution::ConstOffset resolution() const;
    rti::flat::SequenceOffset<Pixel::ConstOffset> pixels() const;
};

class NDDSUSERDllExport CameraImageOffset : public rti::flat::MutableOffset {
public:
    typedef CameraImageConstOffset ConstOffset;
    // Const accessors
    const rti::flat::StringOffset source() const;
    Format format() const;
    Resolution::ConstOffset resolution() const;
    rti::flat::SequenceOffset<Pixel::ConstOffset> pixels() const;
    
    // Modifiers
    rti::flat::StringOffset source();
    bool format(Format value);
    Resolution::Offset resolution();
    rti::flat::SequenceOffset<Pixel::Offset> pixels();
};

```

要创建可变大小的data-samples，应用程序使用Builders。Builder类型提供了创建可变样本成员的接口。

Builders提供了三种类型的函数：

- add_\<member\> 插入一个final类型的成员，并返回一个指向它的Offset。
- build_\<member\> 提供另一个Builder来创建一个mutable类型的成员。
- finish和finish_sample 分别结束一个成员或一个sample的构造。

```c++
class NDDSUSERDllExport CameraImageBuilder : public rti::flat::AggregationBuilder {
public:
    typedef CameraImageOffset Offset;
    Offset finish();
    CameraImage * finish_sample();
    rti::flat::StringBuilder build_source();
    bool add_format(Format value);
    Resolution::Offset add_resolution();
    rti::flat::FinalSequenceBuilder<Pixel::Offset> build_pixels();
};
```

创建一个FlatData sample。

对于final类型，通过对DataWriter的get_loan函数的调用直接进行创建。DataWriter管理这个sample，并在sample被写入后的某个时刻将其返回到池中。以上面IDL中的Pixel为例子：

```cpp
Pixel *pixel_sample = writer.extensions().get_loan();
```

Pixel_sample包含可以被写入的缓冲区。要设置其值，首先需要定位顶级类型的位置：

```c++
PixelOffset pixel = pixel_sample->root();
```

调用root()函数返回一个PiexlOffset，它指向数据开始的位置。然后进行赋值：

```c++
pixel.red(10);
pixel.green(20);
pixel.blue(30);
```

对于mutable类型，使用Builder来创建。使用build_add函数来获取CameraImageBuilder用来构建CameraImage sample。这个函数从DataWriter中借用必要的内存来创建一个CameraImage样本，并提供一个CameraImageBuilder来填充它。使用Builder函数来设置样本的成员（顺序任意）。这些Builder函数在预分配的缓冲区上工作，它们不会分配任何额外的内存。

```c++
CameraImageBuilder image_builder = rti::flat::build_data(writer);
```

首先，添加成员format。作为一个原始成员，add_format函数直接添加成员并设置其值：

```c++
image_builder.add_format(Format::RGB);
```

接下来，添加成员resolution。由于它的类型为final，add_resolution函数添加了成员并提供了允许设置其值的Offset：

```c++
ResolutionOffset resolution = image_builder.add_resolution();
resolution.height(100);
resolution.width(200);
```

为了构建字符串成员source，build_source函数返回一个StringBuilder。使用这个builder（在这种情况下，它就像调用set_string一样简单），然后调用finish。finish函数完成了成员的构建，并使source_builder无效。

```c++
auto source_builder = image_builder.build_source();
source_builder.set_string(“CAM-1”);
source_builder.finish();
```

要创建pixels成员，需要构建一个Pixel的序列：

```c++
auto pixels_builder = image_builder.build_pixels();
```

有两种方法来填充。

方法1：添加并初始化每一个元素

对于具有final类型元素的序列，Builders提供了add_next函数来添加元素。当元素类型是mutable时，序列(数组)Builders提供了build_next函数，来为每个元素提供一个Builder。

```c++
for (int i = 0; i < 20000; i++) {
PixelOffset pixel = pixels_builder.add_next();
pixel.red(i % 256);
pixel.green((i + 1) % 256);
pixel.blue((i + 2) % 256);
} 
pixels_builder.finish();
```

方法2：将序列中的元素转换为等效的C++ plain类型

首先用Builder函数add_n一次添加20000个元素，但不初始化它们。然后获取成员的Offset，将其转换，并将数据作为plain C++类型进行操作。

```c++
pixels_builder.add_n(20000);
auto pixels_offset = pixels_builder.finish();
auto plain_pixels = rti::flat::plain_cast(pixels_offset);
for (int i = 0; i < 20000; i++) {
    plain_pixels[i].red(i % 256);
    plain_pixels[i].green((i + 1) % 256);
    plain_pixels[i].blue((i + 2) % 256);
}
```

最后，调用finish_sample来获取完整的sample。此后，Builder实例将无效，不能再被使用。

```c++
CameraImage *image_sample = image_builder.finish_sample();
```

一旦样本被创建，只要修改不改变大小，仍然可以修改其值。例如，可以改变现有piexl的值，但是不能添加新的像素：

```c++
auto pixels_offset = image_sample->root().pixels();
pixels_offset.get_element(100).blue(0);
```

写一个FlatData sample。

当使用一个常规的DataWriter（对于一个具有Plain语言绑定的类型）写入一个样本时，DataWriter会在其内部队列中复制样本，所以当write()结束时，应用程序仍然拥有样本。然而，对于FlatData类型的DataWriter，它不复制样本，而是保留一个引用。从调用write()的那一刻起，就放弃了数据样本的所有权。

```c++
writer.write(*image_sample);
```

读一个FlatData sample。

对于FlatData类型，无论是final还是mutable，读取数据的方式都是一样的。

```c++
dds::sub::LoanedSamples<CameraImage> samples = camera_reader.take();
```

然后处理第一个样本，假设samples.length() > 0 并且 samples[0].info().valid()，

```c++
const CameraImage& image_sample = samples[0].data();
```

使用root Offset和成员得Offset，以下示例将打印样本的值。在这个示例中，image_sample是常量，因此camera_image是CameraImageConstOffset，只允许去读缓冲区，而不能修改：

```c++
auto camera_image = image_sample->root();

std::cout << "Source: " << camera_image.source().get_string() << std::endl;
std::cout << "Format: " << camera_image.format() << std::endl;

auto resolution = camera_image.resolution();
std::cout << "Resolution (height: " << resolution.height()
<< ", width: " << resolution.width() << ")\n";

```

访问pixels序列，使用构建它时使用的相同的两种方法：

方法1：访问每一个元素的Offset

```c++
for (auto pixel : camera_image.pixels()) {
    std::cout << "Pixel (" << pixel.red() << ", " << pixel.green()
    << ", " << pixel.blue() << ")\n";
}
```

方法2：plain转换

```c++
auto pixel_count = camera_image.pixels().element_count();
auto plain_pixels = rti::flat::plain_cast(camera_image.pixels());
for (int i = 0; i < pixel_count; i++) {
    const auto& pixel = plain_pixels[i];
    std::cout << "Pixel (" << pixel.red() << ", " << pixel.green()
    << ", " << pixel.blue() << ")\n";
}
```

## Zero Copy Transfer Over Shared Memory

在使用内置的共享内存传输进行同一节点内的通信时，默认情况下，Connext DDS 会将样本复制四次。 FlatData 语言绑定将复制次数减少到两次：一次是将样本复制到发布应用程序中的共享内存段中，另一次是在订阅应用程序中重新组装样本。然而，根据样本大小和系统要求，两次复制可能仍然过多。 

提供作为单独库的 Zero Copy 共享内存传输（nddsmetp）可以将复制减少到零，用于同一主机内的通信。此功能通过使用共享内存（SHMEM）内置传输发送DataWriter的SHMED段内关于Sample的16字节的引用来实现零拷贝，而不是使用SHMEM内置传输通过复制来发送序列化的sample内容。

使用 Zero Copy 共享内存传输时，DataWriter 无需对样本进行序列化，而 DataReader 无需对传入的样本进行反序列化，因为样本直接在 DataWriter 创建的 SHMEM 段上访问。过程如下：

1. DataWriter从SHMEM池中获取sample，并向应用程序提供对应的指针；
2. DataWriter对sample进行写操作；
3. DataWrter将sample关联的引用发送到内置的SHMEM传输中；
4. DataReader接收sample的关联应用，并向应用程序提供对应的指针。

![image5](image5.png)

此功能具有如下特点：

- 减少拷贝次数，从四次减少到零次。不再通过多次拷贝来传输整个sample，而只需要将共享内存中的位置分发给DataReaders。
- 由于减少了拷贝次数，内存消耗和CPU负载对应减少。
- 传输延时不再与传输sample的大小相关。
- 使用共享内存的零拷贝传输时，不需要进行分段，因为DataReaders和DataWriters交换的是SHMEM引用（仅16字节），而不是整个sample。

### Send data with Zero Copy transfer over shared memory

使用关键字@transfer_mode(SHMEM_REF)在IDL中对需要使用零拷贝通信传输的类型进行标注，如果该类型不是固定大小的类型，还需要使用关键字@language_binding(FLAT_DATA)和@mutable进行标注。如下是一个示例：

```
enum Format {
RGB,
HSV,
YUV
};

struct Resolution {
long height;
long width;
};

const long IMAGE_SIZE = 8294400 * 3;

@transfer_mode(SHMEM_REF)
struct CameraImage {
long long timestamp;
Format format;
Resolution resolution;
octet data[IMAGE_SIZE];
};
```

零拷贝传输模式，允许应用程序从零拷贝DataWriter创建的共享内存sample池中写入样本。因此在创建sample前先创建一个DataWriter：

```c++
const int MY_DOMAIN_ID = 0;
dds::domain::DomainParticipant participant(MY_DOMAIN_ID);
dds::topic::Topic<CameraImage> camera_topic(participant, "Camera");
dds::pub::DataWriter<CameraImage> camera_writer(
rti::pub::implicit_publisher(participant),
camera_topic);
```

使用DataWriter的get_loan()API从共享内存中获取sample：

```c++
CameraImage *camera_image = camera_writer->get_loan();
```

按照正常sample的方式填充字段：

```c++
camera_image->timestamp(12345678);
camera_image->format(Format::HSV);
camera_image->resolution().height(1024);
camera_image->resolution().width(2048);
```

使用常规的写操作来写入sample：

```c++
camera_writer.write(*camera_image);
```

### Receive data with Zero Copy transfer over shared memory

通过DataReader的take()读取sample：

```c++
dds::sub::LoanedSamples<CameraImage> samples = camera_reader.take();
```

处理第一个sample（假设**samples.length() > 0** and **samples[0].info().valid()**）：

```c++
const CameraImage& camera_image_sample = samples[0].data();
// Process the sample
process_data(camera_image_sample);
if (!camera_reader->is_data_consistent(camera_image_sample)) {
    // Sample was overwritten, ignore this sample
    rollback(camera_image_sample);
}
```

### Check data consistency with Zero Copy transfer over shared memory

共享内存的零拷贝不会进行任何拷贝，在订阅应用程序中处理的sample实际上位于DataWriter的发送队列中。发布应用程序中的DataWriter可以在原始sample被DataReader处理之前或者处理时决定重用此内存以发送不同的样本，这会导致数据一致性问题。

可靠的DataWriter在sample未被确认时不会尝试重用该内存。通过可靠的通信和应用程序级别的确认，订阅应用程序可以通过延迟确认直到sample被处理后来阻住程序重用sample。

如果应用程序的DataWriter和DataReader未同步，没有应用程序级别的同步机制，订阅应用程序可以使用DataReader的is_data_consistent() API来检测数据不一致性，前提是类型不是FlatData。如果类型是FlatData，则在DataWriter重用样本时读取数据样本的行为是未定义的。

如果类型不是FlatData，为了使is_data_consistent() 正常工作，需要将DataWriter的对应QoS，设置writer_qos.transfer_mode.shmem_ref_settings.enable_data_consistency_check为true（默认）。这样DataWriter将会与每个样本关联的特殊序列号做为内联QoS（元数据）发送，DataReader可以使用这些序列号来检查样本在DataWriter处的有效性。简单来说，该API会检查共享内存空间是否已被重用，如果是，则数据就不一致。

is_data_consistent()API只是有助于检测数据不一致性，而不能防止它。推荐的使用方法如下：

```c++
process(data);
if (! reader->is_data_consistent(data, sample_info))
	discard(processed_data);
```

## 测试案例

设计两个DDS应用模拟发送摄像机获取的数据，CameraImage_publisher和CameraImage_subscriber分别包含一个DataWriter和DataReader。CameraImage_publisher首先发送ping包，其中包含了当前发送的时间，CameraImage_subscriber在接收到该包后，立即发送pong包。计算时间间隔，对比使用零拷贝传输以及正常的传输方式之间的延迟时间。

其中plain代表了普通的传输方式，flat代表了使用flat语言绑定，zero_zopy代表了使用零拷贝共享内存传输。

![image6](image6.png)

### 延时对比：

**PC**

在IDL中设置IMAGE_SIZE = 8294400 * 3

![image7](image7.png)

![image8](image8.png)

![image9](image9.png)

在IDL中设置IMAGE_SIZE = 8294400 * 6

![image10](image10.png)

![image11](image11.png)

![image12](image12.png)

**G9Q**

在IDL中设置IMAGE_SIZE = 500000 * 1

![image13](image13.png)

![image14](image14.png)

![image15](image15.png)

在IDL中设置IMAGE_SIZE = 500000 * 2

![image16](image16.png)

![image17](image17.png)

![image18](image18.png)