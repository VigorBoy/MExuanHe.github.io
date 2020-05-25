---
title: AVFoundation 文本转语音和音频录制 播放
date: 2020-05-26 00:30:11
tags:
---
现在你应该对`AVFoundation`有了比较深入的了解，并且对数字媒体的细节也有了一定认识，下面介绍一下 `AVFoundation`的文本转语音功能

# **AVSpeechSynthesizer**
开发者可以使用`AVFoundation`中的`AVSpeechSynthesizer`类向iOS应用程序中添加类似功能，这个类用来播放一个或多个语音内容，这些语音内容都是名为`AVSpeechUtterance`的类的实例。具体的实现代码如下所示：
``` Swift
let synthesizer = AVSpeechSynthesizer()
synthesizer.speak(AVSpeechUtterance(string: "Hello AV Foundation. How are you?"))
```
就两行代码解决了文本转语音功能。当然很多人会有自己的需求，那么还需要对具体对话中用到的声音和语音字符串定义属性。

###### AVSpeechUtterance
如果我们想定义声音的语速和语种那么我们就需要用`AVSpeechUtterance`类来设置一下自定义的属性
``` Swift
let synthesizer = AVSpeechSynthesizer()
//如果播放语音内容 需要用到 AVSpeechUtterance 类
let utterance = AVSpeechUtterance(string: "Hello AV Foundation. How are you?")
//定义播放的语音语种
utterance.voice = AVSpeechSynthesisVoice(language: "en-US")
//定义播放语音内容的速率
utterance.rate = 0.5
//可在播放特定语句时改变声音的音调 pitchMultiplier 的允许值一般介于0.5（低音调）和2.0（高音调）之间
utterance.pitchMultiplier = 1.0
//让语音合成器在播放下一语句之前有短暂时间暂停
utterance.postUtteranceDelay = 0.5
//播放
synthesizer.speak(utterance)
```

强调一下`AVSpeechUtterance`中的`voice`属性
```
Arabic (ar-SA)
Chinese (zh-CN, zh-HK, zh-TW)
Czech (cs-CZ)
Danish (da-DK)
Dutch (nl-BE, nl-NL)
English (en-AU, en-GB, en-IE, en-US, en-ZA)
Finnish (fi-FI)
French (fr-CA, fr-FR)
German (de-DE)
Greek (el-GR)
Hebrew (he-IL)
Hindi (hi-IN)
Hungarian (hu-HU)
Indonesian (id-ID)
Italian (it-IT)
Japanese (ja-JP)
Korean (ko-KR)
Norwegian (no-NO)
Polish (pl-PL)
Portuguese (pt-BR, pt-PT)
Romanian (ro-RO)
Russian (ru-RU)
Slovak (sk-SK)
Spanish (es-ES, es-MX)
Swedish (sv-SE)
Thai (th-TH)
Turkish (tr-TR)
```
`AVSpeechSynthesizer` 常用的`delegate`
``` Swift
  //开始朗读
func speechSynthesizer(_ synthesizer: AVSpeechSynthesizer, didStart utterance: AVSpeechUtterance) {
    }
    
    //结束朗读
func speechSynthesizer(_ synthesizer: AVSpeechSynthesizer, didFinish utterance: AVSpeechUtterance) {
        
    }
    
    //暂停朗读
func speechSynthesizer(_ synthesizer: AVSpeechSynthesizer, didPause utterance: AVSpeechUtterance) {
        
    }
    
    //继续朗读
func speechSynthesizer(_ synthesizer: AVSpeechSynthesizer, didContinue utterance: AVSpeechUtterance) {
        
    }
    
    //将要播放的语音文字
func speechSynthesizer(_ synthesizer: AVSpeechSynthesizer, willSpeakRangeOfSpeechString characterRange: NSRange, utterance: AVSpeechUtterance) {
        
    }
```
常用的文本转语音功能介绍完了 接下来介绍下常用的音频录制和播放功能


**所有iOS应用程序都具有音频会话，无论其是否使用。默认音频会话来自于以下一些预配置：**

- 激活了音频播放，但是音频录音未激活
- 当用户切换响铃/静音开光到“静音”模式时，应用程序播放的所有音频都会消失
- 当设备显示解锁屏幕时，应用程序的音频处于静音状态
- 当应用程序播放音频时，所有后台播放的音频都会处于静音状态

`AVFoundation`定义了7种分类来描述应用程序所使用的音频行为。

|分类|作用|是否允许混音|音频输入|音频输出|
|:---|:---:|:---:|:---:|:---:|
|Ambient|游戏 效率应用程序|是|否|是|
|Solo Ambient (默认)|游戏 效率应用程序|否|否|是|
|Playback|音频和视频播放器|可选|否|是|
|Record|录音机  音频捕捉|否|是|否|
|Play and Record|VoIP 语音聊天|可选|是|是|
|Audio Processing|离线会话和处理|否|否|否|
|Multi-Route|使用外部硬件的高级A/V应用程序|否|是|是|

上述分类所提供的几种常见行为可以满足大部分应用程序的需要，不过如果开发者需要更复杂的功能，其中一些分类可以通过使用`options和modes`方法进一步自定义开发。

**音频会话在应用程序的生命周期中是可以修改的，但通常我们只对其配置一次，就是在应用程序启动时。**
``` Swift

    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        
        let session = AVAudioSession.sharedInstance()
        do {
            /*AVAudioSession.Category:
             .ambient         混合播放，会把后台播放的音乐混合起来播放
             .soloAmbient     进入后台，先会把之前的后台音乐停止，在播放自己的
             .playback        进入后台的时候播放音乐 不会随着静音键和屏幕关闭而静音
             .record          用于需要录音的应用,除了来电铃声，闹钟或日历提醒之外的其它系统声音都不会被播放
             .playAndRecord   用于既需要播放声音又需要录音的应用 该Category提供录音和播放功能。如果你的应用需要用到iPhone上的听筒，该category是你唯一的选择，在该Category下声音的默认出口为听筒（在没有外接设备的情况下）
             .audioProcessing 主要用于音频格式处理，一般可以配合AudioUnit进行使用
             .multiRoute      这个类别可以支持多个设备输入输出。
             
             
             上面介绍的这个七大类别，可以认为是设定了七种主场景，而这七类肯定是不能满足开发者所有的需求的。CoreAudio提供的方法是，首先定下七种的一种基调，然后在进行微调。CoreAudio为每种Category都提供了些许选项来进行微调。在设置完类别后，可以通过 AVAudioSession.CategoryOptions属性 查看当前类别设置了哪些选项
             AVAudioSession.CategoryOptions:
             .mixWithOthers     是否可以和其他后台APP进行混音
             .duckOthers        是否压低其他APP声音
             .allowBluetooth    是否支持蓝牙耳机
             .defaultToSpeaker  是否默认用免提声音
             除此之外，在iOS9还提供了.interruptSpokenAudioAndMixWithOthers iOS10又新加了两个.allowBluetoothA2DP 和 .allowAirPlay用来支持蓝牙A2DP耳机和AirPlay
             
             
             通过上面的七大类别，我们基本覆盖了常用的主场景，在每个主场景中可以通过Option进行微调。为此CoreAudio提供了七大比较常见微调后的子场景。叫做各个类别的模式。
             AVAudioSession.Mode:
             .default        每种类别默认的就是这个模式，所有要想还原的话，就设置成这个模式。
             .voiceChat      主要用于VoIP场景，此时系统会选择最佳的输入设备，比如插上耳机就使用耳机上的麦克风进行采集。此时有个副作用，他会设置类别的选项为".allowBluetooth"从而支持蓝牙耳机。  适用于 .playAndRecord
             .gameChat       适用于游戏App的采集和播放，比如“GKVoiceChat”对象，一般不需要手动设置 适用于 .playAndRecord
             .videoRecording 录制视频时  适用于 .playAndRecord .record
             .measurement    最小系统    适用于 .playAndRecord .record .playback
             .moviePlayback  视频播放    适用于 .playback
             .videoChat      主要用于视频通话，比如QQ视频、FaceTime。时系统也会选择最佳的输入设备，比如插上耳机就使用耳机上的麦克风进行采集并且会设置类别的选项为".allowBluetooth" 和 ".defaultToSpeaker"。 适用于 .playAndRecord
            */
            try session.setCategory(.playback, mode: .default, options: .mixWithOthers)
            
            try session.setActive(true, options: .notifyOthersOnDeactivation)
        } catch let error as Error {
            print(error)
        }

        return true
    }
```

# **使用AVAudionPlayer 播放音频**
`AVAudioPlayer`构建于`Core Audio`中的`C-based Audio Qucue Serics`的最顶层。所以它可以提供所有你在 `Audio Oucue Services`中所能找到的核心功能， 比如播放、循环甚至音频计量。除非你需要从网络流中播放音频、需要访问原始音频样本或者需要非常低的时延，否则`AVAudioPlayer`都能胜任。

#### 创建 AVAudionPlayer
``` Swift

            let fileurl = Bundle.main.url(forResource: "rock", withExtension: "mp3")
            
            let player = try AVAudioPlayer.init(contentsOf: fileurl!)
            
            if player != nil {
                player.prepareToPlay()
            }
```

如果返回一个有效的播放实例，建议开发者调用  `prepareToPlay` 方法。这样做会取得需要的音频硬件并预加载` Audio Queue` 的缓冲区。调用 `prepareToPlay`这个动作是可选的，当调用`Play`方法时会隐形激活，不过在创建时准备播放器可以降低调用`Play`方法和听到声音之间的延时

`AVAudioPlayer`常用属性
``` Swift
            //设置声音的大小 范围为（0到1）
            player?.volume = 0.5
            //设置循环次数，如果为负数，就是无限循环
            player?.numberOfLoops = -1
            //设置播放进度
            player?.currentTime = 0
            //调整播放率 范围 0.5 - 2.0
            player?.rate = 0.5
            //允许使用立体声播放声音 范围从 -1.0（极左）到 1.0（极右） 默认值为0.0（居中）
            player?.pan = 1.0

```

`pause`和`stop`方法的区别：
`pause`和`stop`方法在应用程序外面看来实现的功能都是停止当前播放行为，这两者最主要的区别在底层处理上。掉用`stop`方法会撤销掉用`prepareToPlay`时所做的设置，而调用`pause`方法则不会。

# **使用AVAudionRecorder 播放音频**
`AVAudionRecorder`同其于播放音频的兄弟类一样，构建于`Audio Qucue Serics`之上，是一个功能强大且代码简单易用的类。我们可以在`Mac`机器和`iOS`设备上使用这个类来从内置的麦克风录制视频，也可从外部音频设备进行录制，比如数字音频接口或USB麦克风

#### 创建 AVAudionRecorder

``` Swift
        let tmpDir = NSTemporaryDirectory()
        
        let fileURL = URL.init(string: tmpDir)!.appendingPathComponent("memo.caf")
        
                      /*AVEncoderAudioQualityKey：声音质量 需要的参数是一个枚举 ：
                        AVAudioQualityMin    最小的质量
                        AVAudioQualityLow    比较低的质量
                        AVAudioQualityMedium 中间的质量
                        AVAudioQualityHigh   高的质量
                        AVAudioQualityMax    最好的质量
                        AVLinearPCMBitDepthKey：比特率  8 16 32
                    */ 
        let settings = [AVFormatIDKey : [kAudioFormatAppleIMA4],
                        AVSampleRateKey : "44100.0",
                        AVNumberOfChannelsKey : "1",
                        AVEncoderBitDepthHintKey : "16",
                        AVEncoderAudioQualityKey : [kRenderQuality_Medium]
                        ] as [String : Any]
        
        
        do {
            
            try recorder = AVAudioRecorder.init(url: fileURL, settings: settings)
            
            recorder?.delegate = self
            recorder?.isMeteringEnabled = true
            recorder?.prepareToRecord()
        } catch _ {
            
        }
``` 
成功创建`AVAudioRecorder` 实例，建议调用期` prepareToRecord` 方法，与`AVPlayer`的`prepareToPlay`方法类似。这个方法执行底层Audio Queue初始化的必要过程。该方法还在URL参数指定的位置一个文件，将录制启动时的延迟降到最小。

**在设置字典中指定的键值信息也值得讨论一番，开发者可以使用的完整可用键信息在`<ACFoundation/AVAudioSettings.h>`中定义。大部分的键都专门定义了特有的各式，不过下面介绍的都是一些通用的音频格式**

**1.音频格式**
`AVFormatIDKey` 键定义了写入内容的音频格式，下面的常量都是音频格式所支持的值：

    kAudioFormatLinearPCM               = 'lpcm',
    kAudioFormatAC3                     = 'ac-3',
    kAudioFormat60958AC3                = 'cac3',
    kAudioFormatAppleIMA4               = 'ima4',
    kAudioFormatMPEG4AAC                = 'aac ',
    kAudioFormatMPEG4CELP               = 'celp',
    kAudioFormatMPEG4HVXC               = 'hvxc',
    kAudioFormatMPEG4TwinVQ             = 'twvq',
    kAudioFormatMACE3                   = 'MAC3',
    kAudioFormatMACE6                   = 'MAC6',
    kAudioFormatULaw                    = 'ulaw',
    kAudioFormatALaw                    = 'alaw',
    kAudioFormatQDesign                 = 'QDMC',
    kAudioFormatQDesign2                = 'QDM2',
    kAudioFormatQUALCOMM                = 'Qclp',
    kAudioFormatMPEGLayer1              = '.mp1',
    kAudioFormatMPEGLayer2              = '.mp2',
    kAudioFormatMPEGLayer3              = '.mp3',
    kAudioFormatTimeCode                = 'time',
    kAudioFormatMIDIStream              = 'midi',
    kAudioFormatParameterValueStream    = 'apvs',
    kAudioFormatAppleLossless           = 'alac',
    kAudioFormatMPEG4AAC_HE             = 'aach',
    kAudioFormatMPEG4AAC_LD             = 'aacl',
    kAudioFormatMPEG4AAC_ELD            = 'aace',
    kAudioFormatMPEG4AAC_ELD_SBR        = 'aacf',
    kAudioFormatMPEG4AAC_ELD_V2         = 'aacg',    
    kAudioFormatMPEG4AAC_HE_V2          = 'aacp',
    kAudioFormatMPEG4AAC_Spatial        = 'aacs',
    kAudioFormatAMR                     = 'samr',
    kAudioFormatAMR_WB                  = 'sawb',
    kAudioFormatAudible                 = 'AUDB',
    kAudioFormatiLBC                    = 'ilbc',
    kAudioFormatDVIIntelIMA             = 0x6D730011,
    kAudioFormatMicrosoftGSM            = 0x6D730031,
    kAudioFormatAES3                    = 'aes3',
    kAudioFormatEnhancedAC3             = 'ec-3',
    kAudioFormatFLAC                    = 'flac',
    kAudioFormatOpus                    = 'opus'


指定`kAudioFormatLinearPCM`会将未压缩的音频流写入到文件中。这种格式的保真度最高，不过相应的文件也最大。选择诸如`AAC`或`Apple IMA4`的压缩格式会显著缩小文件，还能保证高质量的音频内容

**2.采样率**
`AVSampleRateKey`用于定义录音器的采样率，采样率定义了对输入的模拟音频信号每一秒内的采样数。在录制音频的质量及最终文件大小方面，采样率扮演着至关重要的角色。使用低采样率，比如`8kHz`， 会导致粗粒度、 AM广播类型的录制效果，不过文件会比较小，使用`44.1kHz`的采样率(CD质量的采样率)会得到非常高质量的内容，不过文件就比较大。对于使用什么采样率最好 没有一个明确的定义，不过开发者应该尽量使用标准的采样率，比如`8000`、`16000`、`22 050`或`44 100`。最终是我们的耳朵在进行判断。

**3.通道数**
`AVNumberOfChannelsKey`用于定义记录音频内容的通道数。指定默认值1意味着使用单声道录制，设置为2意味着使用立体声录制。除非使用外部硬件进行录制，否则通常应该创建单声道录音。

**4.指定格式的键**
处理`Linear PCM`或压缩音频格式时，可以定义一些其他指定格式的键。 可在Xcode帮助文档中的`AV Foundation Audio Sttings Constants`引用中找到完整的列表。

**`AVAudionRecorder`常用的API**

``` Swift
open func prepareToRecord() -> Bool    准备录音， 这个在录音之前要调用一下， 底层可能会给你做一些事

open func record() -> Bool    录音

@available(iOS 6.0, *)
open func record(atTime time: TimeInterval) -> Bool    在未来的某个时刻开始录音

open func record(forDuration duration: TimeInterval) -> Bool  录音， 控制时长

@available(iOS 6.0, *)
open func record(atTime time: TimeInterval, forDuration duration: TimeInterval) -> Bool  在未来的某段时间录多少时间的音频

open func pause()  暂停

open func stop()  停止

open func deleteRecording() -> Bool  删除录音

open func updateMeters()   更新音量等数据

open func peakPower(forChannel channelNumber: Int) -> Float  最高音量

open func averagePower(forChannel channelNumber: Int) -> Float  平均音量

open var isRecording: Bool { get }   是否正在录音

open var url: URL { get }  录音存放的url

open var settings: [String : Any] { get } 录音格式配置字典

@available(iOS 10.0, *)
open var format: AVAudioFormat { get }  录音格式配置

unowned(unsafe) open var delegate: AVAudioRecorderDelegate? 代理

open var currentTime: TimeInterval { get }  当前录音的时长

@available(iOS 6.0, *)
open var deviceCurrentTime: TimeInterval { get }  设备当前时间

@available(iOS 7.0, *)
open var channelAssignments: [AVAudioSessionChannelDescription]? 每个声音通道描述数组

``` 

**`AVAudioRecorderDelegate`**

``` Swift
@available(iOS 3.0, *)
optional public func audioRecorderDidFinishRecording(_ recorder: AVAudioRecorder, successfully flag: Bool)  录音成功的回调

@available(iOS 3.0, *)
optional public func audioRecorderEncodeErrorDidOccur(_ recorder: AVAudioRecorder, error: Error?) 录音发生错误的的回调

@available(iOS, introduced: 2.2, deprecated: 8.0)
optional public func audioRecorderBeginInterruption(_ recorder: AVAudioRecorder)    录音开始中断的回调

@available(iOS, introduced: 6.0, deprecated: 8.0)
optional public func audioRecorderEndInterruption(_ recorder: AVAudioRecorder, withOptions flags: Int) 录音结束中断的回调

```

# 使用Audio Metering
**`AVAudioRecorder`和`AVAudioPlayer`中最强大和最实用的功能就是对音频进行测量。`Audio Metering`可让开发者读取音频的平均分贝和峰值分贝数据，并使用这些数据以可视化方式将声音的大小呈现给最终用户。**

这两个类使用的方法都是**`averagePowerForChannel`**和**`peakPowerForChannel`**。两个方法都会返回一个用于表示声音分贝（dB）等级的浮点值。这个值的范围从表示最大分贝的0Db(fullscale)到表示最小分贝或静音的-160dB。

在可以读取这些值之前，首先要通过设置录音器的**`isMeteringEnabled = true`**才可以支持对音频进行测量。这就使得录音器可以对捕捉到的音频样本进行分贝计算。每当需要读取值时，首页需要调用**`updateMeters()`**方法才能获取最新的值。

创建一个新的CADisplayLink 定时器 来实时刷新 获取分贝 定时器方法内部实现：
``` Swift
       recorder?.updateMeters()
       var level = 0.0
        
        //获取分贝
        let peakPower = recorder?.peakPower(forChannel: 0)
        
        //设置一个最低分贝
        let minDecibels:Float = -60
        
        if peakPower < minDecibels {
            level = 0.0
        }else if peakPower >= 0.0 {
            level = 1.0
        }else {
            let root            = 2.0;
            //最小分贝的波形幅度值 公式: db 分贝 A 波形幅度值   dB=20∗log(A)→A=pow(10,(db/20.0))
            let minAmp          = powf(10.0, 0.05 * minDecibels);
            let inverseAmpRange = 1.0 / (1.0 - minAmp);
            //实时获取到波形幅度值 公式同上
            let amp             = powf(10.0, 0.05 * peakPower);
            //(实时数据 - 最小数据)/(最大数据 - 最小数据)  应该计算的一个比例值吧
            let adjAmp          = (amp - minAmp) * inverseAmpRange;
            level = powf(adjAmp, 1.0 / root);
        }
```
本章我们见识了`AVFoundation`的音频类所能提供的强大功能。`AVAudionSession`作为应用程序和更在的iOS音频环境的中间环节，可通过使用分类在语义上定义应用程序的行为，并且提供工具来观察中断和线路变化。`AVAudionPlayer`和`AVAudioRecorder`提供了一种简单但功能强大的接口，用于处理音频的播放和录制。这两个类都构建与`Core Audio`框架之上，但为在应用程序中实现音频录制和播放提供了一种更便捷的方法。