```
diff --git a/services/core/java/com/android/server/DeviceIdleController.java b/services/core/java/com/android/server/DeviceIdleController.java
index ba3edb7..e52a1ea 100755
--- a/services/core/java/com/android/server/DeviceIdleController.java
+++ b/services/core/java/com/android/server/DeviceIdleController.java
@@ -100,6 +100,8 @@ import java.io.PrintWriter;
 import java.nio.charset.StandardCharsets;
 import java.util.Arrays;
 
+import android.os.SystemProperties;
+
 /**
  * Keeps track of device idleness and drives low power mode based on that.
  */
@@ -2303,6 +2305,7 @@ public class DeviceIdleController extends SystemService
 
     void scheduleAlarmLocked(long delay, boolean idleUntil) {
         if (DEBUG) Slog.d(TAG, "scheduleAlarmLocked(" + delay + ", " + idleUntil + ")");
+        delay = shortenScheduledAlarmIfNeeded(delay);
         if (mMotionSensor == null) {
             // If there is no motion sensor on this device, then we won't schedule
             // alarms, because we can't determine if the device is not moving.  This effectively
@@ -2320,6 +2323,21 @@ public class DeviceIdleController extends SystemService
         }
     }
 
+    //darview++
+    long shortenScheduledAlarmIfNeeded(long toShortenDelay){
+	String needToShortenAlarm;
+	long finalDelay=toShortenDelay;
+	needToShortenAlarm = SystemProperties.get("persist.asus.shortenAlarm", "false");
+	if(needToShortenAlarm.equals("true") && (toShortenDelay == 30*60*1000L)){
+	    Slog.e(TAG,"shortenalarm,we need to shorten alarm");
+	    finalDelay =2*60*1000L;
+	}else{
+	    Slog.e(TAG,"shortenalarm,no need to shorten alarm");
+	}
+	return finalDelay;
+    }
+    //darview--
+
     void scheduleLightAlarmLocked(long delay) {
         if (DEBUG) Slog.d(TAG, "scheduleLightAlarmLocked(" + delay + ")");
         mNextLightAlarmTime = SystemClock.elapsedRealtime() + delay;
@@ -2542,7 +2560,7 @@ public class DeviceIdleController extends SystemService
         pw.println("    Force to be inactive, ready to freely step idle states.");
         pw.println("  unforce");
         pw.println("    Resume normal functioning after force-idle or force-inactive.");
-        pw.println("  get [light|deep|force|screen|charging|network]");
+        pw.println("  get [light|deep|force|screen|charging|network|shortenalarm]");
         pw.println("    Retrieve the current given state.");
         pw.println("  disable [light|deep|all]");
         pw.println("    Completely disable device idle mode.");
@@ -2558,6 +2576,8 @@ public class DeviceIdleController extends SystemService
         pw.println("    Print packages that are temporarily whitelisted.");
         pw.println("  tempwhitelist [-u] [package ..]");
         pw.println("    Temporarily place packages in whitelist for 10 seconds.");
+        pw.println("  shortenalarm true/false");
+        pw.println("    shorten deep alarm from 30m to 2m whille delay=30m");
     }
 
     class Shell extends ShellCommand {
@@ -2695,6 +2715,12 @@ public class DeviceIdleController extends SystemService
                             case "screen": pw.println(mScreenOn); break;
                             case "charging": pw.println(mCharging); break;
                             case "network": pw.println(mNetworkConnected); break;
+                            //darview_test
+                            case "shortenalarm":
+                                        String needShortenAlarm = SystemProperties.get("persist.asus.shortenAlarm", "false");
+                                        pw.print("needShortenAlarm=");
+                                        pw.println(needShortenAlarm);
+                                        break;
                             default: pw.println("Unknown get option: " + arg); break;
                         }
                     } finally {
@@ -2704,7 +2730,34 @@ public class DeviceIdleController extends SystemService
                     pw.println("Argument required");
                 }
             }
-        } else if ("disable".equals(cmd)) {
+        }else if("shortenalarm".equals(cmd)){
+            getContext().enforceCallingOrSelfPermission(android.Manifest.permission.DEVICE_POWER,
+                    null);
+            synchronized (this) {
+                long token = Binder.clearCallingIdentity();
+                String oldShorten = SystemProperties.get("persist.asus.shortenAlarm", "false");
+                String newShorten = oldShorten;
+                String arg = shell.getNextArg();
+                if (arg != null){
+                    try {
+                        if("true".equals(arg) || "false".equals(arg)){
+                            SystemProperties.set("persist.asus.shortenAlarm", arg);
+                            newShorten = SystemProperties.get("persist.asus.shortenAlarm", "false");
+                        }else{
+                            pw.println("wrong argument,the argument should be true/false");
+                        }
+                        pw.print("oldshorten=");
+                        pw.println(oldShorten);
+                        pw.print("newshorten=");
+                        pw.println(newShorten);
+                    }finally {
+                        Binder.restoreCallingIdentity(token);
+                    }
+                }else{
+                    pw.println("Argument required");
+                }
+            }
+        }else if ("disable".equals(cmd)) {
             getContext().enforceCallingOrSelfPermission(android.Manifest.permission.DEVICE_POWER,
                     null);
             synchronized (this) {
@@ -3001,6 +3054,7 @@ public class DeviceIdleController extends SystemService
 
             pw.print("  mLightEnabled="); pw.print(mLightEnabled);
             pw.print("  mDeepEnabled="); pw.println(mDeepEnabled);
+            pw.print("  shortenAlarm="); pw.print(SystemProperties.get("persist.asus.shortenAlarm", "false"));
             pw.print("  mForceIdle="); pw.println(mForceIdle);
             pw.print("  mMotionSensor="); pw.println(mMotionSensor);
             pw.print("  mCurDisplay="); pw.println(mCurDisplay);
```
