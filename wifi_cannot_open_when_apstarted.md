# 1.问题
### 1.复现
关闭wifi(后台扫描开),打开热点,打开wifi,发现wifi打不开(921693)

原因:

WifiStateMachien中mOperationalMode不正确  
被设置为SCAN_ONLY_WITH_WIFI_OFF_MODE而不是CONNECT_MODE,导致wifi关闭,并转向ScanModeState  
**根本原因**是WifiController中状态机转换错误

解决方法:改变目标状态

### 2.相关代码

```java

1.CMD_SET_AP
2.CMD_WIFI_TOGGLED
3.CMD_AP_STOPPED
WifiController.ApEnabledState.processMessage

case CMD_WIFI_TOGGLED:
    if (mSettingsStore.isWifiToggleEnabled()) {
        Slog.d(TAG,"apenabledstate,,CMD_WIFI_TOGGLED");
        if (mStaAndApConcurrency) {
            deferMessage(obtainMessage(msg.what, msg.arg1,  1, msg.obj));
            transitionTo(mApStaEnabledState);
        } else {
            mWifiStateMachine.setHostApRunning(null, false);
            //mPendingState = mStaEnabledState;
            "mPendingState = mDeviceActiveState;"
        }
}
break;

case CMD_SET_AP:
    if (msg.arg1 == 0) {
        Slog.d(TAG,"apenabledstate,,CMD_SET_AP");
        if (mStaAndApConcurrency) {
            mSoftApStateMachine.setHostApRunning(null, false);
        } else {
            mWifiStateMachine.setHostApRunning(null, false);
            "mPendingState = getNextWifiState();"
        }
    }
    break;

case CMD_AP_STOPPED:
    if (mPendingState == null) {
        /**
        * Stop triggered internally, either tether notification
        * timed out or wifi is untethered for some reason.
        */
        mPendingState = getNextWifiState();
    }
    if (mPendingState == mDeviceActiveState && mDeviceIdle) {
        checkLocksAndTransitionWhenDeviceIdle();
    } else {
        // go ahead and transition because we are not idle or we are not going
        // to the active state.
        transitionTo(mPendingState);
    }
break;
```

```java
WifiController.ApEnabledState.getNextWifiState
private State getNextWifiState() {
    "if (mSettingsStore.getWifiSavedState() == WifiSettingsStore.WIFI_ENABLED)" {
        "return mDeviceActiveState;"
    }

    if (mSettingsStore.isScanAlwaysAvailable()) {
        return mStaDisabledWithScanState;
    }

    return mApStaDisabledState;
}
```

