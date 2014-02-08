```
 # copy to path
cp -i ~/Downloads/adt-bundle-mac-x86_64-20131030/sdk/platform-tools/adb /usr/local/bin/
 # see which devices are available (emulators, phones)
adb devices -l
 # query the Intents for the HTC One
adb -s HT36JW912632 shell dumpsys package > /tmp/junk.pkg
```
