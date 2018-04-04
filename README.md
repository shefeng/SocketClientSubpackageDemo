# SocketClientSubpackageDemo
## 这是ios端socket接受服务器分包数据合包的demo

### 以下是ios端接受服务器分包数据并合包的核心代码

``` OC

//
//  ViewController.m
//  SocketClientSubpackageDemo
//
//  Created by 佘峰 on 2018/4/4.
//  Copyright © 2018年 佘峰. All rights reserved.
//

#import "ViewController.h"
#import "GCDAsyncSocket.h"

#define SERVER_IP @"192.168.0.104"
#define SERVER_PORT 12345

@interface ViewController()<GCDAsyncSocketDelegate>{
    
    NSMutableData *_buffer;
    GCDAsyncSocket *clientSocket;
}

@property (weak, nonatomic) IBOutlet UIImageView *testIV;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    
    _buffer = [NSMutableData data];
    
    clientSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:dispatch_get_main_queue()];
    
    NSError *error = nil;
    
    [clientSocket connectToHost:SERVER_IP onPort:SERVER_PORT viaInterface:nil withTimeout:-1 error:&error];
    
    NSLog(@"%@",error);
}


- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
}

#pragma mark - GCDAsyncSocketDelegate
- (void)socket:(GCDAsyncSocket *)sock didAcceptNewSocket:(GCDAsyncSocket *)newSocket{
    NSLog(@"didAcceptNewSocket");
}

//已连接
- (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(uint16_t)port{
    
    NSLog(@"didConnectToHost");
    
    NSLog(@"%@",[NSString stringWithFormat:@"服务器IP: %@-------端口: %d", host,port]);
    
    //主动开始读取数据
    [sock readDataWithTimeout:-1 tag:0];
}

/**
 * Called when a socket has completed reading the requested data into memory.
 * Not called if there is an error.
 **/
- (void)socket:(GCDAsyncSocket *)sock didReadData:(NSData *)data withTag:(long)tag{
    
    NSLog(@"didReadData");
    
    NSUInteger dataLength = data.length;
    NSLog(@"dataLength = %ld",(long)dataLength);//特别注意，此处打印出data的length是为了提醒大家，这里并不是服务器分包发送的4096！！！
    
    [_buffer appendData:data];
    
    do {
        Byte head[8];
        
        [_buffer getBytes:head range:NSMakeRange(0, 8)];//将0-7拷贝到head数组中
        
        //包类型
        int packageType = ((head[3] & 0xFF) << 24) | ((head[2] & 0xFF) << 16) | ((head[1] & 0xFF) << 8) | (head[0] & 0xFF);
        //包大小
        int totalPackageSize = ((head[7] & 0xFF) << 24) | ((head[6] & 0xFF) << 16) | ((head[5] & 0xFF) << 8) | (head[4] & 0xFF);
        
        NSLog(@"================================================================");
        NSLog(@"包类型 = %d",packageType);
        NSLog(@"包大小 = %d",totalPackageSize);
        NSLog(@"================================================================");
        
        if (_buffer.length >= ceilf(totalPackageSize/4088.0) * 4096) {
            //_buffer中已经有一个整包的完整内容了 ------ >搞它
            NSMutableData *imageData = [[NSMutableData alloc] init];
            
            for (int i = 1; i <= ceilf(totalPackageSize/4088.0); i++) {
                int begin = i * 8 + 4088 * (i - 1);
                int len = 4088;
                
                if (i == ceilf(totalPackageSize/4088.0)) {
                    len = totalPackageSize % 4088;
                }
                
                [imageData appendData:[_buffer subdataWithRange:NSMakeRange(begin, len)]];
            }
            
            UIImage *image = [UIImage imageWithData:imageData];
            
            _testIV.image = image;
            
            //这一帧已经用过了，不要了，阉了他
            [_buffer replaceBytesInRange:NSMakeRange(0, 4096 * ceilf(totalPackageSize/4088.0)) withBytes:NULL length:0];
        }else{
            break;
        }
    } while (_buffer.length > 8);
    
    [clientSocket readDataWithTimeout: -1 tag: 0];
    
}

- (void)socket:(GCDAsyncSocket *)sock didReadPartialDataOfLength:(NSUInteger)partialLength tag:(long)tag{
    NSLog(@"didReadPartialDataOfLength");
}

- (void)socket:(GCDAsyncSocket *)sock didWriteDataWithTag:(long)tag{
    NSLog(@"didWriteDataWithTag");
}

- (void)socket:(GCDAsyncSocket *)sock didWritePartialDataOfLength:(NSUInteger)partialLength tag:(long)tag{
    NSLog(@"didWritePartialDataOfLength");
}

- (NSTimeInterval)socket:(GCDAsyncSocket *)sock shouldTimeoutReadWithTag:(long)tag elapsed:(NSTimeInterval)elapsed bytesDone:(NSUInteger)length{
    NSLog(@"shouldTimeoutReadWithTag");
    return 10000;
}

- (NSTimeInterval)socket:(GCDAsyncSocket *)sock shouldTimeoutWriteWithTag:(long)tag elapsed:(NSTimeInterval)elapsed bytesDone:(NSUInteger)length{
    NSLog(@"shouldTimeoutWriteWithTag");
    return 10000;
}

- (void)socketDidCloseReadStream:(GCDAsyncSocket *)sock{
    NSLog(@"socketDidCloseReadStream");
}

- (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(nullable NSError *)err{
    NSLog(@"socketDidDisconnect");
}

- (void)socketDidSecure:(GCDAsyncSocket *)sock{
    NSLog(@"socketDidSecure");
}

- (void)socket:(GCDAsyncSocket *)sock didReceiveTrust:(SecTrustRef)trust completionHandler:(void (^)(BOOL shouldTrustPeer))completionHandler{
    NSLog(@"didReceiveTrust");
    
}


@end


```
