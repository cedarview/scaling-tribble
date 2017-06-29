# 1.WiFi
#### 1.wifi cannot open when ap started and wifi closed before
问题:wifi关闭的情况下,打开热点,再打开wifi,wifi会先打开后自动关闭

原因:WifiController中状态机转换错误  

解决方案:将热点开着时打开wifi后的下一状态正确设置

#### 2. don't auto scan when screen_on & wifiStateMachine was in ScanModeState
问题(964471):亮屏之后wifi没有自动连接,其实是因为没有扫描

原因:  
sleep_policy == 0;息屏一段时间后wifi断开,  
WifiStateMachine退到ScanModeState  
WifiController退至ScanOnlyLockHeldState  
亮屏之后,WifiStateMachine处理消息的时候  
CMD_SCREEN_STATE_CHANGED 来的比 CMD_SET_OPERATIONAL_MODE 早一些,导致无法启动启动扫描的流程.

解决方案:  
在ScanModeState添加对CMD_SCREEN_STATE_CHANGED处理的代码,如果
wifi当前处于关闭状态,则延迟对亮屏消息的处理

#### 3.wifi_icon flash when close airplane_mode or hotspot
问题:关闭热点或飞行模式的时候wifi图标闪

原因:  
关闭热点或飞行模式时,重新打开wifi,有两次设置wifistate  
导致图标闪烁

解决方案:  
第一次设置打开状态时,不发送广播,即不点亮wifi图标

#### 4.hotspot reopen continually & cannot stop  
问题:热点自动重启

原因:  
TetherMasterSM掉到ErrorState的子状态出不来  
在TetherInterfaceSM状态机的InitialState中检查TetherMasterSM上次是否进入ErrorState  
如果是,使TetherMasterSM退回InitialState  
# 2.Doze  
#### 1. whitelisted Twin app do not have network when in doze  
问题:双开的白名单程序不能访问网络  

原因:  
添加白名单程序到iptables时,无法获取到双开应用的userId

解决方案:  
通过提供的接口获取双开应用的userId,再查找所有package,看其是否是双开app,如果是双开app则将其加入要加的iptables的列表中.

# 3.AppStandby
