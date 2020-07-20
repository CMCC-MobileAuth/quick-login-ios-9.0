# 1. 接入指南
sdk技术问题沟通QQ群：609994083</br>

**注意事项：**

1. 一键登录服务必须打开蜂窝数据流量并且手机操作系统给予应用蜂窝数据权限才能使用
2. 取号请求过程需要消耗用户少量数据流量（国外漫游时可能会产生额外的费用）
3. 一键登录服务目前支持中国移动2/3/4G（2,3G因为无线网络环境问题，时延和成功率会比4G低）和中国电信4G（如有更新会在技术沟通QQ群上通知）
4. <font color="red"><strong>（新增）建议全部SDK方法由主线程发起调用，因为SDK的数据容器就是在主线程进行读写操作的，如果外部发起调用SDK方法的线程不是主线程可能会导致多线程对数据容器读写导致崩溃</strong></font>
5. <font color="red"><strong>（新增）SDK不建议嵌套和并发调用，即在上一次请求未结束（未有回调）马上发起下一次请求时，第二次请求将不会有回调，成功率将无法得到保证。</strong></font>
6. 关于双卡的适配问题：
   1. 当两张卡的运营商不一致时，SDK会获取设备上网卡的运营商并进行取号，但上网卡不一定会获取成功（飞行模式状态时），若获取失败，SDK将默认取号卡为移动运营商取号，如果匹配，则取号成功，否则SDK返回103111；
   2. 当SDK存在缓存并且两张卡的运营商不相同时，SDK会重新获取上网卡运营商与上一次取号的运营商进行对比，若两次运营商不一致，则以最新设置的上网卡的运营商为准，重新取号，上次获取的缓存将自动失效；双卡运营商相同的情况则不需要重新取号。
   3. iOS 13上已完成双卡适配，SDK通过苹果提供的方法获取运营商，若获取失败，SDK将默认取号卡为移动运营商取号，如果匹配，则取号成功，否则SDK返回103111.


## 1.1. 接入流程

**1.申请appid和appkey**

根据《开发者接入流程文档》，前往中国移动开发者社区（dev.10086.cn)，按照文档要求创建开发者账号并申请appid和appkey，并填写应用的包名（bundle ID)。

**2.申请能力**

应用创建完成后，在能力配置页面上，勾选应用需要接入的能力类型，如一键登录，并配置应用的服务器出口IP地址。（如果在服务端需要用非对称加密方法对一些重要信息进行加密处理，请在能力配置页面填写RSA加密的公钥）

**3.添加appid白名单**

开发者在完成步骤1和步骤2后，将appid提供给移动认证工作人员，移动方将在2个工作日内将appid加入白名单，白名单添加完毕，开发者就可以开始联调对接。

**4.上线审核**

应用上线前，开发者需要将一键登录取号能力的场景所使用的授权页面（授权页面参考授权页面规范）提供给移动认证产品接口人，审核无误后可正式上线。


## 1.2. 开发流程

**第一步：获取SDK及相关文档**

请在业务联调群中联系移动认证运营获取最新的SDK包和文档

**第二步：搭建开发环境**

1. xcode版本需使用9.0以上，否则会报错
2. 导入认证SDK的framework，直接将移动认证`TYRZSDK.framework`拖到项目中
3. 在Xcode中找到`TARGETS-->Build Setting-->Linking-->Other Linker Flags`在这选项中需要添加`-ObjC`

**第三步：开始使用移动认证SDK**

**[1] 初始化SDK**

在需要进行登录操作的场景进行以下的初始化调用。

```objective-c
[UASDKLogin.shareLogin registerAppId:APPID appKey:APPKEY encrypType:@""];
```

**注意：**开发者不调用该方法注册，SDK核心的API（取号和取token）均不生效，当App侧需要关闭SDK功能时，可不调用该方法

<div STYLE="page-break-after: always;"></div>
**[2] 设置SDK取号或者授权获取token的超时时间（不设置默认8s）**

调用以下方法设置超时（**注：超时时间单位是毫秒**）

```objective-c
[UASDKLogin.shareLogin setTimeoutInterval:10000.f];
```

**注意：**超时设置小于等于0时，SDK均默认超时时间为8s

**小技巧：**如果开发者希望能够单独的给预取号和授权登录请求设置超时时间，可以在调用这俩方法前都分别设置一次这个超时时间即可。

<div STYLE="page-break-after: always;"></div>




# 2. 一键登录功能

## 2.1. 准备工作

在中国移动开发者社区进行以下操作：

1. 获得appid和appkey、APPSecret（服务端）；
2. 勾选一键登录能力；
3. 配置应用服务器的出口ip地址
4. 配置公钥（如果使用RSA加密方式）
5. **针对本机号码校验：**勾选本机号码校验短验辅助开关（可选）
6. <font color="red">商务对接签约（未签约应用使用体验版套餐，每个appid每天只能调用1000次，三个月到期）</font>

## 2.2. 流程说明 

1. 应用发起取号请求，成功后，应用将得到手机号掩码，SDK将缓存取号临时凭证scrip。
2. 开发者按照要求（详见：**2.4章节**）构建授权页面。
3. 用户授权同意后，应用发起授权请求，成功后，应用将得到换号凭证token。
4. 携带token请求获取手机号码接口，获取用户的手机号码信息。

![](https://github.com/CMCC-MobileAuth/quick-login-ios-9.0/blob/master/image/login_process.png?raw=true)

## 2.3. 取号请求

本方法用于发起取号请求，SDK完成网络判断、蜂窝数据网络切换等操作并缓存凭证scrip。<font color="red">缓存允许用户在未开启蜂窝网络时成功取号。</font>

注意：缓存有效时间为60分钟

**请求示例代码**

```objective-c
[UASDKLogin.shareLogin getPhoneNumberCompletion:^(NSDictionary * _Nonnull sender) {
    NSLog(@"result = %@", sender);
}];
```

**取号方法原型**

```objective-c
- (void)getPhoneNumberCompletion:(void (^)(NSDictionary * sender))completion;
```

**completion返回参数**

| 参数          | 类型     | 说明                          |
| ------------- | -------- | ----------------------------- |
| resultCode    | NSString | 返回相应的结果码              |
| desc          | NSString | 调用描述                      |
| securityPhone | NSString | 手机号码掩码，如“138XXXX0000” |
| operatorType  | NSString | 运营商类型：</br>0.未知；</br>1.移动流量；</br>2.联通流量；</br>3.电信流量 |
| traceId       | NSString | 用于定位SDK问题 |
| scripExpiresIn | NSString | 表示scrip有效期，单位：秒 |


## 2.4. 创建授权页

为了确保用户在登录过程中将手机号码信息授权给开发者使用的知情权，一键登录需要开发者提供授权页登录页面供用户授权确认。开发者在调用授权登录方法前，必须弹出授权页，明确告知用户当前操作会将用户的本机号码信息传递给应用。授权页面的设计、布局、生成、弹出和消失，由开发者自行处理，但必须遵守移动认证授权页面设计规范。

注1：如果开发者需要使用**一键登录**服务，必须按照规定创建授权页面

注2：**本机号码校验**不需要创建授权页面，可以直接跳过2.4章

### 2.4.1. 页面规范细则

1、页面必须包含登录/注册按钮，授权登录方法必须绑定该按钮使用。

2、登录按钮文字描述必须包含“登录”或“注册”等文字，<font color="red">不得诱导用户授权。（如果应用需要扩大产品的使用范围，需提前向移动相关接口人进行报备）</font>

3、页面需要提示应用获取到的是用户的本机号码，例如，可以在页面显示本机号码的掩码（139xxxx0000），或者提示用户将使用“<font color="red">本机号码</font>”作为账号登录或注册。

4、页面必须包含移动认证协议条款，其中：
* 移动：
	* 条款名称：《中国移动认证服务条款》
	* 条款页面地址：https://wap.cmpassport.com/resources/html/contract.html 
* 电信：
	* 协议名称：《中国电信天翼账号服务条款》
	* 协议链接：https://e.189.cn/sdk/agreement/detail.do

5、应用在上线前需将满足上述1~4的授权页面（正式上线版的）截图提供给产品接口人审核。

6、应用后续升级时，如果授权页面有较大改动（针对1~4内容进行修改），需将改动的授权页面截图提供给产品接口人审核。

7、对于未遵照1~4设计要求，或通过技术手段故意屏蔽不弹出授权页面但获得调用接口凭证token的行为，能力提供方有权限制APP一键登录取号能力的使用，待整改后再恢复提供服务。

### 2.4.2. 构建授权页控制器

授权登录页面由开发者按照规范设计和构建

1.开发者创建自定义子类CustomAuthViewController，并在该子类中布局UI控件

```objective-c
// CustomAuthViewController.h
@interface CustomAuthViewController : UIViewController

@end

// CustomAuthViewController.m
@implementation CustomAuthViewController

-(void)setupUI{
    
}

@end
```

2.客户端在需要拉起授权页的场景初始化授权页控制器动作

```objective-c
//初始化CustomAuthViewController类（即登录授权页）实例控制器，弹出这个授权页面。
CustomAuthViewController *authVC = [[CustomAuthViewController alloc]init];
[self presentViewController:authVC animated:YES completion:nil];
```

## 2.5. 授权请求

用户调用授权方法，获取取号token

**请求示例代码**

```objective-c
//1.构建授权页控制器

//2.调用取号方法（根据实际需求，也可以放在构建授权页控制前调用）
-(void)getPhonenumber{
    
    [UASDKLogin.shareLogin getPhoneNumberCompletion:^(NSDictionary * _Nonnull sender){
        if ([sender[@"resultCode"] isEqualToString:@"103000"]) {
            NSLog(@"取号成功:%@",sender);
            // 显示手机号码掩码
            self.securityPhoneLable.text = sender[@"securityPhone"];
        } else {
            NSLog(@"取号失败:%@",sender);
            [self dismissViewControllerAnimated:YES completion:nil];
        }
    }];
}

//3.授权登录按钮点击事件，调用授权方法
-(void)authorizeLoginButtonClick{
    
    [UASDKLogin.shareLogin getAuthorizationCompletion:^(NSDictionary * _Nonnull sender) {
        if ([sender[@"resultCode"] isEqualToString:@"103000"]) {
            NSLog(@"授权登录成功:%@",sender);
        } else {
            NSLog(@"授权登录失败:%@",sender);
        }
        [self dismissViewControllerAnimated:YES completion:nil];
    }];
}

//@end
```

**授权方法原型**

```objective-c
- (void)getAuthorizationCompletion:(void (^)(NSDictionary *sender))completion;
```


**completion返回参数**

| 参数          | 类型     | 说明                                                         |
| ------------- | -------- | ------------------------------------------------------------ |
| resultCode    | NSString | 返回相应的结果码                                             |
| token         | NSString | 成功时返回：临时凭证，token有效期2min，一次有效，同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |
| securityPhone | NSString | 手机号码掩码，如“138XXXX0000”                                |
| desc          | NSString | 返回描述                                                     |
| traceId       | NSString | 用于定位SDK问题 |
| scripExpiresIn | NSString | 表示scrip有效期，单位：秒  |
| tokenExpiresIn | NSString | 表示token有效期，单位：秒  |

## 2.6. 获取手机号码（服务端）

详细请开发者查看移动认证服务端接口文档说明。

## 2.7. 本机号码校验（服务端）

详细请开发者查看移动认证服务端接口文档说明。

<div STYLE="page-break-after: always;"></div>



# 3. SDK方法说明

## 3.1. 初始化

用于初始化appId、appKey设置。

**原型**

```objective-c
- (void)registerAppId:(NSString *)appId appKey:(NSString *)appKey encrypType:(NSString *_Nullable)encrypType
```

</br>

**请求参数**

| 参数   | 类型     | 说明        |
| ------ | -------- | ----------- |
| appID  | NSString | 应用的appid |
| appKey | NSString | 应用密钥    |
| encrypType | NSString | 缺省参数，开发者统一填写@"" |


## 3.2. 取号请求

本方法用于发起取号请求，SDK完成网络判断、蜂窝数据网络切换等操作并缓存凭证scrip。

**原型**

```objective-c
- (void)getPhoneNumberCompletion:
		(void (^)(NSDictionary * sender))completion;
```

**completion返回参数**

| 参数          | 类型     | 说明                          |
| ------------- | -------- | ----------------------------- |
| resultCode    | NSString | 返回相应的结果码              |
| desc          | NSString | 调用描述                      |
| securityPhone | NSString | 手机号码掩码，如“138XXXX0000” |
| operatorType  | NSString | 运营商类型：</br>0.未知；</br>1.移动流量；</br>2.联通流量；</br>3.电信流量   |
| traceId       | NSString | 用于定位SDK问题 |
| scripExpiresIn | NSString | 表示scrip有效期，单位：秒  |
| tokenExpiresIn | NSString | 表示token有效期，单位：秒  |

**请求示例代码**

```objective-c
 [UASDKLogin.shareLogin getPhoneNumberCompletion: ^ (NSDictionary * _Nonnull sender) {
        if ([sender[@ "resultCode"] isEqualToString: @"103000"]) {
            NSLog(@ "取号成功:%@", sender);
        } else {
            NSLog(@ "取号失败:%@", sender);
        }
    }];
```

## 3.3. 授权请求

SDK的一键登录接口，获取到的token可以在移动认证服务端获取完整手机号

**原型**

```objective-c
- (void)getAuthorizationCompletion:
		(void (^)(NSDictionary *sender))completion;
```

**completion返回参数**

| 参数          | 类型     | 说明                                                         | 是否必填   |
| ------------- | -------- | ------------------------------------------------------------ | ---------- |
| resultCode    | NSString | 返回相应的结果码                                             | 是         |
| token         | NSString | 成功时返回：临时凭证，token有效期2min，一次有效，同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 | 成功时必填 |
| securityPhone | NSString | 手机号码掩码，如“138XXXX0000”                                |            |
| desc          | NSString | 调用描述                                                     | 否         |
| traceId       | NSString | 用于定位SDK问题 |
| scripExpiresIn | NSString | 表示scrip有效期，单位：秒  |
| tokenExpiresIn | NSString | 表示token有效期，单位：秒  |

**完整一键登录调用示例**

```objective-c
//1.构建授权页控制器

//2.调用取号方法（根据实际需求，也可以放在构建授权页控制前调用）
-(void)getPhonenumber{
    __weak typeof(self) weakSelf = self;
    [UASDKLogin.shareLogin getPhoneNumberCompletion:^(NSDictionary * _Nonnull sender){
        if ([sender[@"resultCode"] isEqualToString:@"103000"]) {
            NSLog(@"取号成功:%@",sender);
            // 显示手机号码掩码
            weakSelf.securityPhoneLable.text = sender[@"securityphone"];
        } else {
            NSLog(@"取号失败:%@",sender);
            [weakSelf dismissViewControllerAnimated:YES completion:nil];
        }
    }];
}

//3.授权登录按钮点击事件，调用授权方法
-(void)authorizeLoginButtonClick{
    __weak typeof(self) weakSelf = self;
    [UASDKLogin.shareLogin getAuthorizationCompletion:^(NSDictionary * _Nonnull sender) {
        if ([sender[@"resultCode"] isEqualToString:@"103000"]) {
            NSLog(@"授权登录成功:%@",sender);
        } else {
            NSLog(@"授权登录失败:%@",sender);
        }
        [weakSelf dismissViewControllerAnimated:YES completion:nil];
    }];
}

//@end
```

## 3.4. 本机号码校验

### 3.4.1. 方法描述

**功能**
该方法用于获取**本机号码校验校验token**

**原型**

```objective-c
- (void)mobileAuthCompletion:(void (^)(NSDictionary *sender))completion;
```

**completion返回参数**

| 参数 | 类型 | 说明 |
| :-: | :-: | :-: |
| resultCode | NSString | 返回码 |
| desc | NSString | 描述 |
| token | NSString | 本机号码校验token |
| traceId       | NSString | 用于定位SDK问题 |
| scripExpiresIn | NSString | 表示scrip有效期，单位：秒  |
| tokenExpiresIn | NSString | 表示token有效期，单位：秒  |



## 3.5. 获取网络状态和运营商类型

本方法用于获取用户当前上网卡的网络环境和运营商

**原型**

```objective-c
@property (nonatomic,readonly) NSDictionary<NSString *, NSNumber *> *networkType;
```

**networkType返回参数**

| 参数        | 类型     | 说明                                                         |
| ----------- | -------- | ------------------------------------------------------------ |
| networkType | NSString | 0.无网络;</br>1.数据流量;</br>2.wifi;</br>3.数据+wifi        |
| carrier     | NSNumber | 运营商类型：</br>0.未知；</br>1.移动流量；</br>2.联通流量；</br>3.电信流量 |

## 3.6. 删除临时取号凭证

本方法用于删除取号方法`getPhoneNumberCompletion`成功后返回的取号凭证scrip

注意：删除临时取号凭证后，下次取号请求将不再使用缓存登录，SDK会再次发起网关请求取号。

**原型**

```objective-c
- (BOOL)delectScrip;
```

**completion返回参数**

| 参数  | 类型 | 说明                                          |
| ----- | ---- | --------------------------------------------- |
| state | BOOL | 删除结果状态，（YES：有缓存，已执行删除，NO：无缓存，不执行删除） |

<div STYLE="page-break-after: always;"></div>





# 4. 返回码说明

## 4.1. SDK返回码

使用SDK时，SDK会在认证结束后将结果回调给开发者，其中结果为JSONObject对象，其中resultCode为结果响应码，103000代表成功，其他为失败。成功时在根据token字段取出身份标识。失败时根据resultCode定位失败原因。

| 错误编号      | 返回码描述                       |
| ------------- | ------------------------------|
| 103000        | 成功                           |
| 103101 | 请求异常                                                  |
| 103102 | 包签名/Bundle ID错误（社区填写的appid和对应的包名包签名必须一致） |
| 103111 | 错误的运营商请求（可能是用户正在使用代理或者运营商判断失败导致）   |
| 103119 | appid不存在                                               |
| 103211 | 其他错误，联系技术支撑解决问题 |
| 103412 | 无效的请求（1.加密方式错误；2.非json格式；3.空请求等） |
| 103414 | 参数校验异常 |
| 103511 | 服务器ip白名单校验失败                                       |
| 103811   | token为空 |
| 103902 | scrip失效（短时间内重复登录）                                |
| 103911 | token请求过于频繁，10分钟内获取token且未使用的数量不超过30个 |
| 104201 | token已失效或不存在（重复校验或失效） |
| 105001 | 联通取号失败 |
| 105002 | 移动取号失败                                               |
| 105003 | 电信取号失败                                               |
| 105012 | 不支持电信取号 |
| 105013 | 不支持联通取号 |
| 105018 | token权限不足（使用了本机号码校验的token获取号码） |
| 105019 | 应用未授权（未在开发者社区勾选能力） |
| 105021 | 当天已达取号限额                                           |
| 105302 | appid不在白名单                                            |
| 105312 | 余量不足（体验版到期或套餐用完） |
| 105313 | 非法请求                                                  |
| 200021        | 数据解析异常                    |
| 200022        | 无网络                         |
| 200023        | 请求超时                       |
| 200025        | 未知错误，一般配合desc分析      |
| 200027        | 未开启数据网络或网络不稳定        |
| 200028        | 网络异常  |
| 200038        | 异网取号网络请求失败   |
| 200048        | 用户未安装sim卡                |
| 200050        | EOF异常（iOS:Socket创建或发送接收数据失败，用户未授权蜂窝权限） |
| 200064        | 服务端返回数据异常                    |
| 200072        | CA根证书认证失败         |
| 200082        | 服务器繁忙              |
| 200086        | ppLocation为空         |
| 200096        | 当前网络不支持取号（WiFi+数据环境下，DNS解析取号地址得到的IP类型与蜂窝数据端口支持的IP类型不匹配导致，建议更换WiFi网络或关闭WiFi重试）|
