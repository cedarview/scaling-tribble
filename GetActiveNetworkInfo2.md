# 1. 使用
- 首先获取ConnectivityManager
- 然后再调用GetActiveNetworkInfo

```
  ConnectivityManager connMgr =
      (ConnectivityManager) getSystemService(Context.CONNECTIVITY_SERVICE);
  NetworkInfo activeInfo = connMgr.getActiveNetworkInfo();
  if (activeInfo != null && activeInfo.isConnected()) {
      wifiConnected = activeInfo.getType() == ConnectivityManager.TYPE_WIFI;
      mobileConnected = activeInfo.getType() == ConnectivityManager.TYPE_MOBILE;
      if(wifiConnected) {
          Log.i(TAG, getString(R.string.wifi_connection));
      } else if (mobileConnected){
          Log.i(TAG, getString(R.string.mobile_connection));
      }
  } else {
      Log.i(TAG, getString(R.string.no_wifi_or_mobile));
  }
```
# 2.调用流
 1. ConnectivityManager
 
    调用ConnectivityService的getActiveNetworkInfo
 ```
     public NetworkInfo getActiveNetworkInfo() {
        try {
            return mService.getActiveNetworkInfo();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
 ```
 2. ConnectivityService.getActiveNetworkInfo
    1. 调用**getUnfilteredActiveNetworkState**
    2. 调用**filterNetworkStateForUid**过滤网络状态
    ```
    public NetworkInfo getActiveNetworkInfo() {
        enforceAccessPermission();
        final int uid = Binder.getCallingUid();
        final NetworkState state = getUnfilteredActiveNetworkState(uid);
        filterNetworkStateForUid(state, uid, false);
        maybeLogBlockedNetworkInfo(state.networkInfo, uid);
        return state.networkInfo;
    }
    ```
 3. getUnfilteredActiveNetworkState 获取网络状态信息
 ```
 private NetworkState getUnfilteredActiveNetworkState(int uid) {
    NetworkAgentInfo nai = getDefaultNetwork();
    final Network[] networks = getVpnUnderlyingNetworks(uid);
    if (networks != null) {
        // getUnderlyingNetworks() returns:
        // null => there was no VPN, or the VPN didn't specify anything, so we use the default.
        // empty array => the VPN explicitly said "no default network".
        // non-empty array => the VPN specified one or more default networks; we use the
        //                    first one.
        if (networks.length > 0) {
            nai = getNetworkAgentInfoForNetwork(networks[0]);
        } else {
            nai = null;
        }
    }
    if (nai != null) {
        return nai.getNetworkState();
    } else {
        return NetworkState.EMPTY;
    }
}
 ```
 4. filterNetworkStateForUid 过滤网络状态
 一些app被限制访问网络则获取不到网络信息
 调用**isNetworkWithLinkPropertiesBlocked**判断该uid是否被限制网络访问
 如果判断为Blocked将调用 **networkInfo.setDetailedState**,会将mState设置为State.DISCONNECTED
 ```
 private void filterNetworkStateForUid(NetworkState state, int uid, boolean ignoreBlocked) {
    if (state == null || state.networkInfo == null || state.linkProperties == null) return;
    if (isNetworkWithLinkPropertiesBlocked(state.linkProperties, uid, ignoreBlocked)) {
        state.networkInfo.setDetailedState(DetailedState.BLOCKED, null, null);
    }
    if (mLockdownTracker != null) {
        mLockdownTracker.augmentNetworkInfo(state.networkInfo);
    }
    // TODO: apply metered state closer to NetworkAgentInfo
    final long token = Binder.clearCallingIdentity();
    try {
        state.networkInfo.setMetered(mPolicyManager.isNetworkMetered(state));
    } catch (RemoteException e) {
    } finally {
        Binder.restoreCallingIdentity(token);
    }
}
 ```
5. isNetworkWithLinkPropertiesBlocked判断该uid是否限制网络访问
 - 一部分规则与datasaver有关，一部分与appidle有关
 - 当进入appid的时候会调用会通过NetworkPolicyService调用onUidRulesChanged来更新这些规则
 - 在构造函数中将mPolicyListener设置给了networkPolicyManager的mConnectivityListener
 `mPolicyManager.setConnectivityListener(mPolicyListener);`
 ```
 private boolean isNetworkWithLinkPropertiesBlocked(LinkProperties lp, int uid,
        boolean ignoreBlocked) {
    // Networks aren't blocked when ignoring blocked status
    if (ignoreBlocked) return false;
    // Networks are never blocked for system services
    if (isSystem(uid)) return false;
    final boolean networkMetered;
    final int uidRules;
    synchronized (mVpns) {
        final Vpn vpn = mVpns.get(UserHandle.getUserId(uid));
        if (vpn != null && vpn.isBlockingUid(uid)) {
            return true;
        }
    }
    final String iface = (lp == null ? "" : lp.getInterfaceName());
    synchronized (mRulesLock) {
        networkMetered = mMeteredIfaces.contains(iface);
        uidRules = mUidRules.get(uid, RULE_NONE);
    }
    boolean allowed = true;
    // Check Data Saver Mode first...
    if (networkMetered) {
        if ((uidRules & RULE_REJECT_METERED) != 0) {
            if (LOGD_RULES) Log.d(TAG, "uid " + uid + " is blacklisted");
            // Explicitly blacklisted.
            allowed = false;
        } else {
            allowed = !mRestrictBackground
                  || (uidRules & RULE_ALLOW_METERED) != 0
                  || (uidRules & RULE_TEMPORARY_ALLOW_METERED) != 0;
            if (LOGD_RULES) Log.d(TAG, "allowed status for uid " + uid + " when"
                    + " mRestrictBackground=" + mRestrictBackground
                    + ", whitelisted=" + ((uidRules & RULE_ALLOW_METERED) != 0)
                    + ", tempWhitelist= + ((uidRules & RULE_TEMPORARY_ALLOW_METERED) != 0)"
                    + ": " + allowed);
        }
    }
    // ...then power restrictions.
    if (allowed) {
        allowed = (uidRules & RULE_REJECT_ALL) == 0;
        if (LOGD_RULES) Log.d(TAG, "allowed status for uid " + uid + " when"
                + " rule is " + uidRulesToString(uidRules) + ": " + allowed);
    }
    return !allowed;
}
 ```
6. setDetailedState
前面判断为blocked的时候就会调用setDetailedState将mState设置为DISCONNECTED
`stateMap.put(DetailedState.BLOCKED, State.DISCONNECTED);`
`state.networkInfo.setDetailedState(DetailedState.BLOCKED, null, null);`
```
public void setDetailedState(DetailedState detailedState, String reason, String extraInfo) {
    synchronized (this) {
        this.mDetailedState = detailedState;
        this.mState = stateMap.get(detailedState);
        this.mReason = reason;
        this.mExtraInfo = extraInfo;
    }
}
```
