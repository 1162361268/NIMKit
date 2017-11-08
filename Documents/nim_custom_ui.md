# 聊天界面排版自定义

## 前言

针对开发者对组件的不同定制需求，云信 iOS UI 组件提供了大量配置可以让开发者便捷的修改或自定义排版。根据定制的深度，大体可以分为两种：

* **聊天气泡的简单布局定制**

关于内置聊天气泡的各种内间距，组件均已提出并组成 `plist` 配置文件供开发者直接设置。开发者不需要关心具体的界面实现代码，只需要在配置文件上改一些间距值，即可进行界面调试。

这种定制适用于开发者满足于内置的消息类型，并不需要对消息气泡的界面布局做出很大改变的情况。

* **聊天界面的深度定制**

有的时候，需要根据具体的消息类型并结合业务逻辑的上下文定制聊天界面，这个时候一个简单的配置文件就不再适用了。UI 组件提供一个全局的排版控制器注入接口 `- (void)registerLayoutConfig:(Class)layoutConfigClass` 来让上层开发者自行注入排版配置器。

排版配置器需要实现 `NIMCellLayoutConfig` 协议。



## NIMMessageCell

UI 组件的消息绘制都是统一由 `NIMMessageCell` 类完成的，因此，了解 `NIMMessageCell` 的大致组成，对排版是很有帮助的。

<img src="https://github.com/netease-im/NIM_Resources/blob/master/iOS/Images/nimkit_cell.jpg" width="550" height="210" />

* 蓝色区域：为具体内容 ContentView，如文字 UILabel ,图片 UIImageView 等。

* 绿色区域：为消息的气泡，具体的内容和气泡之间会有一定的内间距，这里为 contentViewInsets 。

* 紫色区域：为整个 UITableViewCell ，具体的气泡和整个cell会有一定的内间距，这里为 cellInsets 。

* 红色区域：为用户的头像。

在刷新数据时，会调用方法并 `-(void)refresh` 将界面模型 `NIMMessageModel` 传入。

当第一次调用这个方法（即不是复用生成），会调用 `- (void)addContentViewIfNotExist` 方法，根据 `NIMMessageModel` 找到对应的布局配置(如果找不到则按未知类型消息处理)。

Tips：开发者在第一次接入的时候，可能由于协议实现不全或者注入布局配置有误等原因，导致消息在界面上显示为 `未知类型消息`，这个时候可以尝试从 `NIMMessageCell` 的 `- (void)addContentViewIfNotExist` 方法入手调试，查看`NIMMessageModel` 对应的布局配置以及协议的返回值是否正确。


## 聊天组件的注入配置
NIMKit 的聊天组件需要开发者通过注入一系列协议接口来进行聊天相关的排版布局和功能逻辑的扩展。
通过以下四个协议的注入配置，可实现聊天界面的基本设置。

* **NIMSessionConfig** 协议主要定义了消息气泡和输入框相关功能的配置，自定义扩展需要新建一个类去实现该接口。注入配置示例代码如下：

```objc
@interface TestSessionConfig : NSObject<NIMSessionConfig>
@end

@implementation TestSessionConfig

//实现 NIMSessionConfig 的相关代理，并进行自定义扩展
//这里不一一列举

@end

@interface TestSessionViewController : NIMSessionViewController

@property (nonatomic, strong) TestSessionConfig *test_config;

@end

@implementation TestSessionViewController

- (id<NIMSessionConfig>)sessionConfig {
    //返回 nil，则使用默认配置，若需要自定义则自己实现
    return nil;
    //返回自定义的 config，则使用此自定义配置
    //return self.test_config;
}

@end
```
* **NIMCellLayoutConfig** 主要提供聊天消息气泡布局相关配置；在 NIMKit 中既是类也是协议，类比 NSObject，方便实现多继承；开发者自定义扩展时建议最好使用继承方式，方便使用 NIMKit 组件自带的默认布局；具体扩展方式见示例如下：

```objc
@interface TestCellLayoutConfig : NIMCellLayoutConfig<NIMCellLayoutConfig>
@end

@implementation NTESCellLayoutConfig

- (CGSize)contentSize:(NIMMessageModel *)model cellWidth:(CGFloat)width {
    //如果需要自定义，这里添加相关处理，否则使用组件默认父类配置
    return [super contentSize:mode cellWidth:width];
}

//其余接口不一一列举
//...
@end

//确保在页面初始化之前注入 TestCellLayoutConfig 使新的布局生效
//示例代码放在 AppDelegate 中

@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    [[NIMKit sharedKit] registerLayoutConfig:[TestCellLayoutConfig new]];
    //...
}

@end
```
* **NIMKitConfig** 主要提供聊天消息相关的常量配置；开发者自定义时可直接修改该类的属性值，需要注意的是由于涉及界面布局，因此需要进入相关视图之前就进行配置，示例代码如下：

```objc
@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    //这里放在 AppDelegate 里进行配置示例，这里只是举个🌰
    [NIMKit sharedKit].config.leftBubbleSettings.textSetting.font = [UIFont fontWithName:@"Arial" size:15.f];

    //...
}

@end
```
* **NIMKitDataProvider** 主要提供用户消息的配置，开发者可通过新建实现或者继承 NIMKitDataProviderImpl 进行自定义扩展，具体示例代码如下：

```objc
//方式一
@interface TestDataProvider : NSObject<NIMKitDataProvider>
@end

@implementation TestDataProvider
- (NIMKitInfo *)infoByUser:(NSString *)userId
                    option:(NIMKitInfoFetchOption *)option {
      NIMKitInfo *info;
      info = [[NIMKitInfo alloc] init];
      info.infoId = userId;
      info.showName = userId;
      return info;
}
@end

@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    //注入自定义的 dataProvider
    [[NIMKit sharedKit].provider = [TestDataProvider new];
    //...
}

@end

//方式二
@interface TestDataProviderImpl : NIMKitDataProviderImpl
@end

@implementation TestDataProviderImpl
//重写相关接口
//- (NIMKitInfo *)infoByUser:(NSString *)userId option:(NIMKitInfoFetchOption *)option
//- (NIMKitInfo *)infoByTeam:(NSString *)teamId option:(NIMKitInfoFetchOption *)option
@end

@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    //注入自定义的 dataProvider
    [[NIMKit sharedKit].provider = [TestDataProviderImpl new];
    //...
}

@end
```
## 聊天气泡的简单布局定制

消息气泡具体属性

<img src="https://github.com/netease-im/NIM_Resources/blob/master/iOS/Images/nimkit_cell_1.jpg" width="550" height="400" />

<img src="https://github.com/netease-im/NIM_Resources/blob/master/iOS/Images/nimkit_cell_2.jpg" width="550" height="210" />

### <p id="session_title"> 1. 聊天界面标题 </p>
包括聊天页面主标题和子标题更改以及字体和字号设置  

```objc
//继承 NIMSessionViewController

@interface TestSessionViewController : NIMSessionViewController
@end

@implementation TestSessionViewController

- (void)viewdidLoad {
    self.titleLabel.textColor = [UIColor blackColor];
    self.titleLabel.font = [UIFont fontWithName:@"Arial" size:14.f];
    self.subTitleLabel.textColor = [UIColor blackColor];
    self.subTitleLabel.font = [UIFont fontWithName:@"Arial" size:14.f];
}

- (NSString *)sessionTitle{
    return @"主标题";
}

- (NSString *)sessionSubTitle {
    return @"子标题";
}

@end
```

### <p id="session_component"> 2. 聊天气泡具体组件 </p>
#### <p id="component_read"> 1）已读回执配置 </p>
可配置是否显示已读回执；单条消息或者全局均可配置是否显示“已读”

```objc
@interface TestConfig : NSObject<NIMSessionConfig>
@end
@implementation TestConfig
//全局
- (BOOL)shouldHandleReceipt
{
    return NO;
}
//单条
- (BOOL)shouldHandleReceiptForMessage:(NIMMessage *)message
{
    //NIM Demo 支持文字，语音，图片，视频，文件，地址位置和自定义消息都已读，其他的不支持
    NIMMessageType type = message.messageType;
    if (type == NIMMessageTypeCustom) {
        NIMCustomObject *object = (NIMCustomObject *)message.messageObject;
        id attachment = object.attachment;
        
        if ([attachment isKindOfClass:[NTESWhiteboardAttachment class]]) {
            return NO;
        }
    }
    return type == NIMMessageTypeText ||
           type == NIMMessageTypeAudio ||
           type == NIMMessageTypeImage ||
           type == NIMMessageTypeVideo ||
           type == NIMMessageTypeFile ||
           type == NIMMessageTypeLocation ||
           type == NIMMessageTypeCustom;
}
@end
```
【注】这里实现 NIMSessionConfig 协议之后，需要确保<a href="#config">第二步</a>中会话视图控制器的相关注入配置
#### <p id="component_timeStamp"> 2）时间戳配置 </p>
通过实现 NIMKitMessageProvider 相关协议进行时间戳显示与否的配置，以及两条时间戳显示间隔的配置

```objc
@interface TestMessageDataProvider : NSObject<NIMKitMessageProvider>
@end

@implementation TestMessageDataProvider

- (BOOL)needTimetag{
    //返回 YES 表明显示时间戳，否则不显示
}

@end
```
【注】这里实现 NIMSessionConfig 协议之后，需要确保<a href="#config">第二步</a>中会话视图控制器的相关注入配置

```objc
@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    //时间间隔放在合适的位置全局配置，示例放在 AppDelegate
    [NIMKit sharedKit].config.messageInterval = 480;
    //...
}

@end
```
#### <p id="component_avatar"> 3）头像配置 </p>
* 头像显示与否配置

```objc
//实现 NIMCellLayoutConfig 协议，继承 NIMCellLayoutConfig 类
@interface TestCellLayoutConfig : NIMCellLayoutConfig<NIMCellLayoutConfig>
@end

@implementation TestCellLayoutConfig

- (BOOL)shouldShowAvatar:(NIMMessageModel *)model
{
    //进行自定义操作或者显示父类的默认值
}

@end
```
【注】这里实现 NIMCellLayoutConfig 协议之后，需要确保<a href="#config">第二步</a>中相关注入配置

* 头像位置配置

```objc
//实现 NIMCellLayoutConfig 协议，继承 NIMCellLayoutConfig 类
@interface TestCellLayoutConfig : NIMCellLayoutConfig<NIMCellLayoutConfig>
@end

@implementation TestCellLayoutConfig

- (CGFloat)avatarMargin:(NIMMessageModel *)model
{
    //自定义头像距离 NIMMessageCell 边框宽度
}

@end
```
* 点击头像的响应事件
 
```objc
- (BOOL)onTapAvatar:(NSString *)userId{
    //设置 NIMMessageCellDelegate 代理，并重写该方法
}
```

* 长按头像的响应事件

```objc
- (BOOL)onLongPressCell:(NIMMessage *)message
                 inView:(UIView *)view {
    //同上，重写该方法进行自定义操作
}
```
#### <p id="component_nickname"> 4）昵称配置 </p>
* 昵称显示与否配置

```objc
@interface TestCellLayoutConfig : NIMCellLayoutConfig<NIMCellLayoutConfig>
@end

@implementation TestCellLayoutConfig

- (BOOL)shouldShowNickName:(NIMMessageModel *)model {
   //自定义
}

@end
```
* 昵称位置配置

```objc
@interface TestCellLayoutConfig : NIMCellLayoutConfig<NIMCellLayoutConfig>
@end

@implementation TestCellLayoutConfig

- (CGFloat)nickNameMargin:(NIMMessageModel *)model {
   //自定义
}

@end
```
【注】这里实现 NIMCellLayoutConfig 协议之后，需要确保<a href="#config">第二步</a>中相关注入配置
#### <p id="component_retry"> 5）重试按钮配置 </p>

```Objective-C
@interface TestCellLayoutConfig : NIMCellLayoutConfig<NIMCellLayoutConfig>
@end

@implementation TestCellLayoutConfig

- (BOOL)disableRetryButton:(NIMMessageModel *)model {
   //自定义
}

@end
```
【注】这里实现 NIMCellLayoutConfig 协议之后，需要确保<a href="#config">第二步</a>中相关注入配置
#### <p id="component_bubble"> 6）消息气泡配置 </p>
* 气泡布局可更改属性在 NIMKitSetting 类中

|           名称           |               定义              |
|:------------------------:|:-------------------------------:|
|       contentInsets      |  设置消息的 contentView 内间距  |
|         textColor        | 设置消息 contentView 的文字颜色 |
|           font           | 设置消息 contentView 的文字字体 |
|   normalBackgroundImage  |    设置消息普通模式下的背景图   |
| highLightBackgroundImage |    设置消息按压模式下的背景图   |
|    cellBackgroundColor   |            cell 的背景色        |
* 气泡类型
配置见 NIMKitSettings 

|       名称       |         定义         |
|:----------------:|:--------------------:|
|    textSetting   |   文本类型消息设置   |
|   audioSetting   |   音频类型消息设置   |
|   videoSetting   |   视频类型消息设置   |
|    fileSetting   |   文件类型消息设置   |
|   imageSetting   |   图片类型消息设置   |
|  locationSetting | 地理位置类型消息设置 |
|    tipSetting    |   提示类型消息设置   |
|   robotSetting   |  机器人类型消息设置  |
| unsupportSetting | 无法识别类型消息设置 |
| teamNotificationSetting | 群组通知类型消息设置 |
| chatroomNotificationSetting | 聊天室类型消息设置 |
| netcallNotificationSetting | 网络电话类型通知消息设置 |
具体默认设置见 NIMKitConfig，这里不一一列举

* 气泡大小与位置更改
气泡根据发消息者是本人或者他人，位置布局不同，分为 leftBubbleSettings 和 rightBubbleSettings 进行配置，配置方式见<a href = "#config">第二步</a> NIMKitConfig 配置方式

#### <p id="component_event"> 7）点击事件处理 </p>
实现 NIMMessageCellDelegate 代理相关方法

* 点击气泡事件

```objc
- (BOOL)onTapCell:(NIMKitEvent *)event {
    //自定义
}
```
* 长按消息气泡

```objc
- (BOOL)onLongPressCell:(NIMMessage *)message
                 inView:(UIView *)view {
    //自定义
}
```
### <p id="session_content"> 3. 聊天消息历史数据获取以及相关配置 </p>

#### <p id="session_data"> 1）聊天消息配置 </p>
聊天的消息数据源可以自行配置，默认从所属本地会话里抓取消息

```objc
@interface NIMSessionMsgDatasource()

//构造匿名内部类
@property (nonatomic, strong) id<NIMKitMessageProvider> dataProvider;

@end

@implementation NIMSessionMsgDatasource

//从本地或者服务器获取消息

@end
```
#### <p id="session_limit"> 2）抓取条数配置 </p>
单次从服务器抓取消息的条数限制配置；通过 NIMKitConfig 里 messageLimit 属性更改配置

```objc
@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    //示例放在 AppDelegate
    [NIMKit sharedKit].messageLimit = 20;
    //...
}

@end
```
#### <p id="session_autofetch"> 3）自动获取历史记录配置 </p>

可配置进入聊天界面自动是否获取历史消息；通过构造 NIMSessionConfig 对象，实现如下接口进行自定义

```objc
@interface TestSessionConfig : NSObject<NIMSessionConfig>
@end

@implementation TestSessionConfig

- (BOOL)autoFetchWhenOpenSession {
    return NO;
}

@end
```
【注】这里实现 NIMSessionConfig 协议之后，需要确保<a href="#config">第二步</a>中会话视图控制器的相关注入配置

### <p id="session_input"> 4. 输入相关配置 </p>

输入框

<img src="https://github.com/netease-im/NIM_Resources/blob/master/iOS/Images/nimkit_input_view.jpg" width="660" height="300" />

#### <p id="session_input"> 1）输入框布局 </p>
整个 NIMInputBar 的按钮类型可自定义，若不实现如下方法，则按照默认按钮顺序排列

```objc
- (NSArray<NSNumber *> *)inputBarItemTypes{
    //示例包含语音、输入框、表情以及更多按钮
    return @[
               @(NIMInputBarItemTypeVoice),
               @(NIMInputBarItemTypeTextAndRecord),
               @(NIMInputBarItemTypeEmoticon),
               @(NIMInputBarItemTypeMore)
            ];
}

//如果不需要录音则去掉 NIMInputBarItemTypeVoice 配置
- (NSArray<NSNumber *> *)inputBarItemTypes{
    //示例包含语音、输入框、表情以及更多按钮
    return @[
               @(NIMInputBarItemTypeTextAndRecord),
               @(NIMInputBarItemTypeEmoticon),
               @(NIMInputBarItemTypeMore)
            ];
}
```
#### <p id="session_input"> 2）@功能配置 </p>
* @功能开启或者关闭通过实现 NIMSessConfig 如下方法进行自定义

```objc
- (BOOL)disableAt {
    //自定义
}
```

* 开启机器人会话功能
通过如下方法，开启或者关闭@联系人列表中的机器人会话选择

```objc
- (BOOL)enableRobot {
   //自定义
}
```
#### <p id="session_input"> 3）输入框文本输入配置 </p>

|      名称      |            定义            |
|:--------------:|:--------------------------:|
|   placeholder  |       输入框的占位符配置       |
| inputMaxLength | 输入框能容纳的最大字符长度 |
| maxNumberOfInputLines | 输入框最大显示行数配置 |

#### <p id="session_input"> 4）输入添加表情配置 </p>
* 默认 emoji 表情图片以及文案配置
可通过替换 NIMKitEmoticon.bundle 里的 emoji 贴图资源和 emoji.plist 进行配置
* 贴图表情配置
通过实现 NIMSessionConfig 的接口，并实现相关方法如下，若配置为 nil 则没有贴图表情配置

```objc
@interface TestConfig : NIMSessionConfig
@end

@implementation TestConfig

- (NSArray<NIMInputEmoticonCatalog *> *)charlets {
    //自定义
}

@end
```
【注】这里实现 NIMSessionConfig 协议之后，需要确保<a href="#config">第二步</a>中会话视图控制器的相关注入配置

* 自定义贴图点击事件
通过实现 NIMInputActionDelegate 代理相关方法

```objc
- (void)onSelectChartlet:(NSString *)chartletId
                 catalog:(NSString *)catalogId {
    //自定义相关方法
}
```

#### <p id="session_input"> 5）更多菜单配置 </p>
* 更多菜单按钮配置
通过实现 NIMSessionConfig 中如下方法进行配置，若不配置，默认只有三个按钮，分别为拍照、相册、地理位置

```objc
- (NSArray *)mediaItems
{
    //这里给出一个示范
    NSArray *defaultMediaItems = [NIMKit sharedKit].config.defaultMediaItems;
    
    NIMMediaItem *janKenPon = [NIMMediaItem item:@"onTapMediaItemJanKenPon:"
                                     normalImage:[UIImage imageNamed:@"icon_jankenpon_normal"]
                                   selectedImage:[UIImage imageNamed:@"icon_jankenpon_pressed"]
                                           title:@"石头剪刀布"];
    
    NIMMediaItem *fileTrans = [NIMMediaItem item:@"onTapMediaItemFileTrans:"
                                                normalImage:[UIImage imageNamed:@"icon_file_trans_normal"]
                                              selectedImage:[UIImage imageNamed:@"icon_file_trans_pressed"]
                                           title:@"文件传输"];
    
    NIMMediaItem *tip       = [NIMMediaItem item:@"onTapMediaItemTip:"
                                     normalImage:[UIImage imageNamed:@"bk_media_tip_normal"]
                                   selectedImage:[UIImage imageNamed:@"bk_media_tip_pressed"]
                                           title:@"提示消息"];
    
    items = @[janKenPon,fileTrans,tip];
    return [defaultMediaItems arrayByAddingObjectsFromArray:items];
}
```

* 更多菜单点击事件处理
通过重写如下方法，进行相关自定义按钮的点击事件处理

```objc
- (BOOL)onTapMediaItem:(NIMMediaItem *)item {
    //自定义点击事件处理
}
```
### <p id="session_record"> 5. 音频录制与播放 </p>
音频录制和播放与消息气泡以及输入框的附加按钮都相关，这里单独提出来作为一个小节做介绍。
#### <p id = "record_type"> 1）配置 NIMSessionConfig 相关接口 </p>
NIMSession 提供录音相关接口有如下几个，开发者通过实现相关接口，实现自定义需要的配置

|           接口名称           |              功能              |
|:----------------------------:|:------------------------------:|
|          recordType          |            录音类型            |
|     disableAutoPlayAudio     |          音频轮播开关          |
| disableAudioPlayedStatusIcon |        语音未读红点开关        |
|    disableProximityMonitor   | 在贴耳的时候自动切换成听筒模式 |
```objc
@interface TestConfig : NSObject<NIMSessionConfig>
@end
@implementation TestConfig

- (NIMAudioType)recordType {
    return NIMAudioTypeAAC;
}

- (BOOL)disableAutoPlayAudio {
    return NO;
}

- (BOOL)disableAudioPlayedStatusIcon {
    return YES;
}

- (BOOL)disableProximityMonitor{
    return NO;
}
```
【注】这里实现 NIMSessionConfig 协议之后，需要确保<a href="#config">第二步</a>中相关注入配置

#### <p id = "record_max"> 2）录音最大时长配置 </p>
录音时长的配置在 NIMKitConfig 中，可通过如下方式配置

```objc
@implementation TestAppDelegate

- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    //...
    //这里放在 AppDelegate 里进行配置示例
    [NIMKit sharedKit].config.recordMaxDuration = 15.f;
    //...
}
```

#### <p id = "record_send"> 3）录音是否可以发送配置 </p>
实现 NIMSessionViewController 相关接口

```objc
@interface TestSessionViewController : NIMSessionViewController
@end

@implementation TestSessionViewCOntroller
- (BOOL)recordFileCanBeSend:(NSString *)filepath
{
   return YES;
}
```

#### <p id = "record_toast"> 4）录音无法发送提示 Toast 文案配置 </p>
实现 NIMSessionViewController 相关接口

```objc
@interface TestSessionViewController : NIMSessionViewController
@end

@implementation TestSessionViewCOntroller
- (void)showRecordFileNotSendReason
{
    [self.view makeToast:@"录音时间太短" duration:0.2f position:CSToastPositionCenter];
}
```

## 聊天界面的深度定制
如果需要结合一些上下文定制聊天界面，就需要采用深度定制。在进入会话页之前，注入布局布局配置到 `NIMKit` 即可

```objc
//注册 NIMKit 自定义排版配置
[[NIMKit sharedKit] registerLayoutConfig:[NTESCellLayoutConfig class]];
```  

布局配置器可以选择实现 `NIMCellLayoutConfig` 接口所定义的方法，不实现的接口，会采用内置的默认布局参数进行处理。

在很多场景下，只是在特殊消息场景下需要修正一下排版配置，其他情况还是沿用默认配置，因此强烈建议自定义的排版控制器继承内置的排版实现 `NIMCellLayoutConfig` 协议。这样在开发者需要自定义布局的场景下，填入自定义配置，其他情况只需调用 `super` 方法即可。

具体实现逻辑示范见 Demo 中 `NTESCellLayoutConfig` 类。



