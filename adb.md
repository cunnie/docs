Prerequisite: you have installed Android Studio

```
 # copy to path
cp -i ~/Library/Android/sdk/platform-tools/adb /usr/local/bin/
 # see which devices are available (emulators, phones)
adb devices -l
 # query the Intents for the Nexus 5
adb -s 04a17276251cf25d shell dumpsys package > /tmp/junk.pkg
adb -s emulator-5554 shell pm list permissions
```
