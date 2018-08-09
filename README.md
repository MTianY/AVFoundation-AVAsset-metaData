## AVFoundation 中的资源和源数据 

### 1.资源的含义?

AVFoundation 中最重要的类就是 `AVAsset`,他是 AVFoundation 设计的核心,在几乎所有特性和功能的开发中扮演着至关重要的角色.

`AVAsset`是一个抽象类和不可变类,定义了媒体资源混合呈现的方式,将媒体资源的静态属性模块化成一个整体,如它们的标题、时长和元数据等.

- AVAsset 提供了对基本媒体格式的层抽象
    - 这意味着无论是处理 QuickTime 影片、MPEG-4视频还是 MP3 音频,对你和对框架而言,面对的只有`资源`这个概念.
    - 这就是我们开发者在面对不同格式的内容时有了一个统一的处理方法,不需要考虑多种编解码器和容器格式因为细节不同而带来的困扰.
    - 不过如果仍希望获取这些信息,也可以通过其他方式实现.

- AVAsset 隐藏了资源的位置信息.
    - 当我们对一个现有媒体对象进行处理时,会通过 URL 进行初始化来创建一个资源.
    - 这个 URL 可能是指向应用程序 bundle 的本地 URL, 也可能是文件系统的其他位置, 也可能是从用户 iPod 库中得到的 URL, 甚至可能是服务器上的一个音频流或视频流的 URL.

`AVAsset`本身并不是媒体资源,但是它可以作为时基媒体的容器.

- 它由一个或多个带有描述自身元数据的媒体组成.
- 我们使用`AVAssetTrack`类代表保存在资源中的统一类型媒体,并对每个资源建立相应的模型.
- `AVAssetTrack`最长见的形态就是音频和视频流,但是它还可以用于表示文本、副标题或隐藏字幕等媒体类型

资源的曲目可通过其 tracks 属性进行访问.对该属性的请求会返回一个 NSArray, 该数组中的元素就是专辑包含的所有曲目.
此外, AVAsset 还可以通过标识符、媒体类型或媒体特征等信息找到相应的曲目.  

### 2.创建资源

当为一个现有媒体资源创建 AVAsset 对象时,可以通过 URL 对它进行初始化来实现.

```objc
NSURL *assetURL = [NSURL urlWithString:@""];
AVAsset *asset = [AVAsset assetWithURL:assetURL];
```

- `AVAsset`是一个抽象类,意味着它不能直接被实例化.
- 当使用它的`assetWithURL:`方法创建实例时,实际上是创建了它子类的一个实例,子类名为`AVURLAsset`
- 有时我们会直接使用`AVURLAsset`这个类,因为它允许通过传递选项字典来精细调整资源的创建方式.
    - 比如,如果创建一个用在音频或视频编辑场景中的资源,可能希望传递一个选项(option)来告诉程序提供更精细的时长和计时信息.

    ```objc
    NSURL *assetURL = [NSURL urlWithString:@""];
    NSDictionary *options = @{ AVURLAssetPreferPreciseDurationAndTimingKey : @YES };
    AVURLAsset *asset = [[AVURLAsset alloc] initWithURL:assetURL options:options];
    ``` 

### 3.异步载入

`AVAsset`具有多种的有用的方法和属性,可以提供有关资源的信息,如

- 时长
- 创建日期
- 元数据

`AVAsset`还包含一些用来获取和使用曲目集合的方法.
`AVAsset`使用一种高效的设计方法,即延时载入资源的属性,直到请求时才载入.这样就可以快速的创建资源,而不用考虑因为立即载入相关媒体或元数据所带来的问题.

**不过有一点很重要:就是属性的访问总是同步发生的**.

- 如果正在请求的属性没有预先载入,程序就会阻塞,直到其可以做出适当的响应,这就会带来卡顿的问题

**`AVAsset`和`AVAssetTrack`都采用了`AVAsynchronousKeyValueLoading`协议.该协议通过下面给出的这些方法实现了异步查询属性的功能:**

```objc
/**
 * 作用: 查询一个给定属性的状态.
 * 该方法会返回一个枚举类型的 AVKeyValueStatus 值,用来表示当前所请求属性的状态.
 * 如果状态不是 AVKeyValueStatusLoaded. 意味着此时请求该属性可能导致程序卡顿.
 */
- (AVKeyValueStatus)statusOfValueForKey:(NSString *)key error:(NSError *)outError

/**
 * 作用: 异步载入一个给定的属性.
 * 为其提供一个具有一个或多个 key 的数组(资源的属性名)和一个 completionHandler 块,当资源处于回应请求的状态时,就会调用这个 completionHandler 块.
 */
- (void)loadValuesAsynchronouslyForKeys:(NSArray *)keys completionHandler:(void(^)(void))handler
```    

### 4.媒体元数据

 通过 AVFoundation 使我们开发者不需要考虑大多数特定格式的细节.不需要我们对相应格式读写操作的底层技术有所了解.就可以使用元数据.
 在处理媒体元数据方面, AVFoundation 提供了一套统一的方法.

#### 4.1 元数据格式

虽然存在多种格式的媒体资源,但是在 Apple 环境下遇到的媒体类型主要有4种,分别是

- QuickTime(mov)
- MPEG-4 video(mp4和 m4v)
- MPEG-4 audio(m4a)
- MPEG-Layer III audio (mp3)

**1.QuickTime**

QuickTime 是由苹果公司开发的一种功能强大、跨平台的媒体架构.该架构的一部分是`QuickTime File Format`规范,定义了`.mov`文件的内部结构.

- QuickTime 文件由一种称为`atoms`的数据结构组成.
    - 一个`atom`包含了描述媒体资源某一方面的数据,或者包含其他`atom`,但不可以两者都包含.
    - `atom`以一种复杂的树状结构组合在一起,详细的对布局、音频样本格式、视频帧信息乃至需要呈现的元数据信息(如作者信息和版权信息)做了描述.

### 4.媒体元数据

 通过 AVFoundation 使我们开发者不需要考虑大多数特定格式的细节.不需要我们对相应格式读写操作的底层技术有所了解.就可以使用元数据.
 在处理媒体元数据方面, AVFoundation 提供了一套统一的方法.

#### 4.1 元数据格式

虽然存在多种格式的媒体资源,但是在 Apple 环境下遇到的媒体类型主要有4种,分别是

- QuickTime(mov)
- MPEG-4 video(mp4和 m4v)
- MPEG-4 audio(m4a)
- MPEG-Layer III audio (mp3)

**1. QuickTime**

QuickTime 是由苹果公司开发的一种功能强大、跨平台的媒体架构.该架构的一部分是`QuickTime File Format`规范,定义了`.mov`文件的内部结构.

- QuickTime 文件由一种称为`atoms`的数据结构组成.
    - 一个`atom`包含了描述媒体资源某一方面的数据,或者包含其他`atom`,但不可以两者都包含.
    - `atom`以一种复杂的树状结构组合在一起,详细的对布局、音频样本格式、视频帧信息乃至需要呈现的元数据信息(如作者信息和版权信息)做了描述.
    
**2. MPEG-4 音频和视频**

MPEG-4 Part 14是定义 MP4 文件格式的规范.
MP4直接派生于 QuickTime 文件格式,这就意味着它与 QuickTime 文件的结构是类似的.
MP4文件也由成为 atom 的数据结构组成.技术上讲, MPEG-4规范讲这些称为 boxes.

- `.mp4`是对 MPEG-4媒体的标准扩展,但存在一些变化,如`.m4v、 .m4a、 .m4p 和.m4b`
- `M4V`文件是带有苹果公司`针对FairPlay`加密及`AC3-audio`
扩展的 MPEG-4视频格式.如果不涉及到`FairPlay 和 AC3-audio`,那么`MP4和 M4V`只是扩展名不同而已.
- `M4A`专门针对音频,使用变化的扩展名的目的是让使用者知道该文件只带有音频资源.
- `M4P`是苹果较旧的 iTunes 音频格式,使用其 FairPlay 扩展.
- `M4B`用于有声读物.并通常包含章节标签及提供书签功能,让读者可以返回到指定位置开始阅读.

**3. MP3**

MP3 文件与上面有很大区别,它不使用`容器格式`,而是使用`编码音频数据`, 包含的可选元数据的结构块通常位于文件开头.

- MP3文件使用一种称为`ID3v2`的格式来保存关于音频内容的描述信息, 包含的数据有歌曲演唱者、所属唱片和音乐风格等.

AVFoundation 支持读取 ID3v2 标签的所有版本,但不支持写入. MP3格式受到专利限制,所以 AVFoundation 无法支持对 MP3或 ID3数据进行编码.


### 5. 使用元数据

**5.1 .可以实现查询相关元数据功能的类:**

- AVAsset
    - 多数情况下使用 
- AVAssetTrack
    - 只有当涉及获取曲目一级元数据等情况时使用

**5.2 .读取具体资源元数据的接口**

- 由 AVMetadataItem 类提供

**5.3 .元数据键空间**

- AVAsset 和 AVAssetTrack 提供了两种方法可以获取相关的元数据,要了解这两个不同的方法额适用范围,首先要知道键空间(key spaces)的含义.
- AVFoundation 使用键空间作为将相关键组合在一起的方法,可以实现对 AVMetadataItem 实例集合的筛选.
- 每个资源最少包含两个键空间,供从中获取元数据.



**5.4 .查找元数据**

当得到一个包含元数据的数组时,通常希望找到所需的具体元数据值.

- 一个特别有效的方法是使用 `AVMetadataItem` 提供的便利方法,来获取结果集合并对其进行筛选.
