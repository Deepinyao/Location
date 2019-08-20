我来简单说一下这几个类的作用：

LocationTracker & Other
和我们直接打交道的主要就是LocationTracker这个类。用这个类，我们可以配置定位的相关参数。我们来看看这个类的主要方法：

+ (CLLocationManager *)sharedLocationManager;
 

构造方法，获得一个LocationTraker的单例对象（不了解单列是啥意思的，你可以理解成创建一个全局变量）。

1 - (void)startLocationTracking;
 

这个方法是开始追踪定位，之后，定位功能就跑起来了。

- (void)stopLocationTracking;
 

这个方法和上面的方法是一对，它用来关闭定位追踪。

- (void)updateLocationToServer;
 

这个方法用来向服务器发送已获取的设备位置信息。 

另外还有两个类是“LocationShareModel”和“BackgroundTaskManager”。他们的工作主要是处理定位服务的后台运行和处理设备获取的定位数据。具体的原理我们不用去管它。
首先我们把我们要用到的类先从下载的项目文件夹中拿出来，我们要用的总共有三个类 ：“LocationTracker”“LocationShareModel”和“BackgroundTaskManager”如下图：



 

下一步，Xcode打开我们要使用这个类库的工程，把这三个类库加入到工程中去（你可以选中这6个文件拖进文件导航）



 

 

抛开这个类库不谈，如果要进行后台定位服务，你需要确保为工程做出如下设置：

1.开启后台定位模式：选中工程Target->Capabilities->Background Modes-勾选Location updates:

 

2.在Plist中添加前/后台定位的键值：在Plist根目录新建两个键值如下，这些键值将会在程序开启时让用户允许开启后前/台定位。



 

设置完以上配置之后，我们就可以来想用我们的voyageLocation啦

 

首先在你想要使用定位功能的ViewController 导入头文件

 

#import "LocationTracker.h"
然后声明两个成员变量：

@property LocationTracker * locationTracker;
@property (nonatomic) NSTimer* locationUpdateTimer;
 

 

之后写一个方法配置LocationTraker：

复制代码
 1 -(void)setUpLocationTraker{
 2     self.locationTracker = [LocationTracker sharedLocationManager];
 3     [self.locationTracker startLocationTracking];
 4     //设定向服务器发送位置信息的时间间隔
 5     NSTimeInterval time = 300.0;
 6     //开启计时器
 7     self.locationUpdateTimer =
 8     [NSTimer scheduledTimerWithTimeInterval:time
 9                                      target:self
10                                    selector:@selector(updateLocation)
11                                    userInfo:nil
12                                     repeats:YES];
13 }
复制代码
 

上面计时器每隔300s运行一次“updateLocation”方法，该方法的实现如下：

1 -(void)updateLocation {
2     NSLog(@"开始获取定位信息...");
3     //向服务器发送位置信息
4     [self.locationTracker updateLocationToServer];
5 }
 

上面的updateLocationToServer方法就是你向服务器发送信息的方法了，这个方法需要你依照自己的需求进行改动打开“LocationTraker.m”文件找到该方法：

复制代码
 1 - (void)updateLocationToServer {
 2     
 3     NSLog(@"updateLocationToServer");
 4     
 5     // Find the best location from the array based on accuracy
 6     NSMutableDictionary * myBestLocation = [[NSMutableDictionary alloc]init];
 7     
 8     for(int i=0;i<self.shareModel.myLocationArray.count;i++){
 9         NSMutableDictionary * currentLocation = [self.shareModel.myLocationArray objectAtIndex:i];
10         
11         if(i==0)
12             myBestLocation = currentLocation;
13         else{
14             if([[currentLocation objectForKey:ACCURACY]floatValue]<=[[myBestLocation objectForKey:ACCURACY]floatValue]){
15                 myBestLocation = currentLocation;
16             }
17         }
18     }
19     NSLog(@"My Best location:%@",myBestLocation);
20     
21     //If the array is 0, get the last location
22     //Sometimes due to network issue or unknown reason, you could not get the location during that  period, the best you can do is sending the last known location to the server
23     if(self.shareModel.myLocationArray.count==0)
24     {
25         NSLog(@"Unable to get location, use the last known location");
26 
27         self.myLocation=self.myLastLocation;
28         self.myLocationAccuracy=self.myLastLocationAccuracy;
29         
30     }else{
31         CLLocationCoordinate2D theBestLocation;
32         theBestLocation.latitude =[[myBestLocation objectForKey:LATITUDE]floatValue];
33         theBestLocation.longitude =[[myBestLocation objectForKey:LONGITUDE]floatValue];
34         self.myLocation=theBestLocation;
35         self.myLocationAccuracy =[[myBestLocation objectForKey:ACCURACY]floatValue];
36     }
37     
38     NSLog(@"Send to Server: Latitude(%f) Longitude(%f) Accuracy(%f)",self.myLocation.latitude, self.myLocation.longitude,self.myLocationAccuracy);
39     
40     //TODO: 在这里插入你向服务器发送请求的代码
41     
42     //当你向服务器发送位置信息成功后，要清空当前的数组，以便下一回合的定位
43     [self.shareModel.myLocationArray removeAllObjects];
44     self.shareModel.myLocationArray = nil;
45     self.shareModel.myLocationArray = [[NSMutableArray alloc]init];
46 }
复制代码
 

在上面代码的第处40行进行修改，添加你像服务器发送位置信息的请求，当请求成功后，不要忘记执行第43-45行的代码，清空数组，以便下一次定位。

例如我加入的代码如下，我是用了AFNetworking的网络请求类库：

复制代码
 1 AFHTTPRequestOperationManager *manager=[AFHTTPRequestOperationManager manager];
 2     NSString *url=[NSString stringWithFormat:@"http://172.1.1.36:8080/uploadDeviceLocation.action"];
 3     NSMutableDictionary *parameter=[[NSMutableDictionary alloc]init];
 4     [parameter setObject:@"####################" forKey:@"udid"];
 5     [parameter setObject: [NSString stringWithFormat:@"%f",self.myLocation.longitude] forKey:@"x"];
 6     [parameter setObject:[NSString stringWithFormat:@"%f",self.myLocation.latitude] forKey:@"y"];
 7     [manager GET:url parameters:parameter success:^(AFHTTPRequestOperation *operation, id responseObject) {
 8         NSLog(@" 成功了");
 9     } failure:^(AFHTTPRequestOperation *operation, NSError *error) {
10         NSLog(@"失败了");
43     [self.shareModel.myLocationArray removeAllObjects];
44     self.shareModel.myLocationArray = nil;
45     self.shareModel.myLocationArray = [[NSMutableArray alloc]init];
11 }];
复制代码
 OK~ 搞定，赶紧试试吧！ 哦对了，你不觉得你忘记什么了吗？ 对了 要把  [self setUpLocationTraker] 方法放到你的 viewDidLoad 里面~哈哈

这样 后台位置上传就解决了。这是控制台打出的Log。

