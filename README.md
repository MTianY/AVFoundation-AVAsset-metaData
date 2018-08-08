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


