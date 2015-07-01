# Introduction #


## Creating A SIMBL Plugin Bundle ##

Creating a SIMBL plugin project is pretty simple, but there are a few things that haven't really ever been properly documented.

  1. Create a new "Cocoa Bundle" project in XCode.
  1. Create a basic plugin class - say, `MySamplePlugin`. You will use this class as a jumping-off point to setup the rest of your hacks.
  1. Edit the `Info.plist`
    * Set `NSPrincipalClass` to `MySamplePlugin`
    * Create a new array key `SIMBLTargetApplications`
    * Create a child dictionary of `SIMBLTargetApplications` with the keys `BundleIdentifier`, `MaxBundleVersion` and `MinBundleVersion`.  All of them should have string values.
      * `BundleIdentifier` should match that of your target application - say `com.apple.Terminal`
      * `MaxBundleVersion` should be a string version of the maximum value of `CFBundleVersion` for your target application. You can just put the current version to be safe.
      * `MinBundleVersion` should be a string version of the minimum value of `CFBundleVersion` for your target application. Again, you can just put the current version.
    * Currently, all of the version numbers must be parse-able as integers, but this may change in the future.  These keys get more relevant when you start sharing your plugin with others. Basically, they are a way of keeping your plugin from loading inside a version of the application that you haven't had a chance to test with. If SIMBL encounters an application with an out-of-bounds version, it politely tells the user with an alert message that it won't load the plugin and to contact the developer.
```
<key>SIMBLTargetApplications</key>
<array>
	<dict>
		<key>BundleIdentifier</key>
		<string>com.apple.Safari</string>
		<key>MaxBundleVersion</key>
		<string>412</string>
		<key>MinBundleVersion</key>
		<string>412</string>
	</dict>
</array>
```
  1. Build out your plugin class.  You probably want to put most of your initialization code in the `load` class method. This is a special method that SIMBL looks for when it loads a plugin's principal class. `load` gets called at a nice "safe" time when the application has mostly initialized itself and is pretty much ready for interaction with the user. While you can put your initialization code in other methods, I recommend putting it here. Usually I make the plugin class a singleton object, since it's often nice to have a single location to call "home."
```
@implementation MySamplePlugin

/**
 * A special method called by SIMBL once the application has started and all classes are initialized.
 */
+ (void) load
{
	MySamplePlugin* plugin = [MySamplePlugin sharedInstance];
	// ... do whatever
	NSLog(@"MySamplePlugin installed");
}

/**
 * @return the single static instance of the plugin object
 */
+ (MySamplePlugin*) sharedInstance
{
	static MySamplePlugin* plugin = nil;

	if (plugin == nil)
		plugin = [[MySamplePlugin alloc] init];

	return plugin;
}
```


# Debugging and Troubleshooting #

If you need to see more information about SIMBL injection, you can enable debug logging by setting some preferences.

## Enabling debug logging ##
```
defaults write net.culater.SIMBL SIMBLLogLevel -int 0
```

## Sample Log Ouput ##
In Console.app you should see something like this:
```
3/18/10 11:54:16 AM	SIMBL Agent[160]	Safari started
3/18/10 11:54:16 AM	SIMBL Agent[160]	app start notification: {
    NSApplicationBundleIdentifier = "com.apple.Safari";
    NSApplicationName = Safari;
    NSApplicationPath = "/Applications/Safari.app";
    NSApplicationProcessIdentifier = 7424;
    NSApplicationProcessSerialNumberHigh = 0;
    NSApplicationProcessSerialNumberLow = 237626;
    NSWorkspaceApplicationKey = <NSRunningApplication: 0x200236840 (com.apple.Safari - 7424)>;
}
3/18/10 11:54:16 AM	SIMBL Agent[160]	checking bundle /Users/mike/Library/Application Support/SIMBL/Plugins/PithHelmet.bundle
3/18/10 11:54:16 AM	SIMBL Agent[160]	checking target identifier com.apple.Safari
3/18/10 11:54:16 AM	SIMBL Agent[160]	send inject event
3/18/10 11:54:17 AM	Safari[7424]	load SIMBL plugins
3/18/10 11:54:17 AM	Safari[7424]	SIMBL loaded by path /Applications/Safari.app <com.apple.Safari>
3/18/10 11:54:17 AM	Safari[7424]	checking bundle /Users/mike/Library/Application Support/SIMBL/Plugins/PithHelmet.bundle
3/18/10 11:54:17 AM	Safari[7424]	checking target identifier com.apple.Safari
3/18/10 11:54:17 AM	PithHelmet[7424]	PithHelmet installed
3/18/10 11:54:17 AM	Safari[7424]	loaded /Users/mike/Library/Application Support/SIMBL/Plugins/PithHelmet.bundle
```

This shows some of the interaction between the SIMBLAgent, the SIMBL OSAX and the actual target plugin (in this case, PithHelmet).

## Disabling Debug Logging ##

The default log level is 2 - this only logs in the case of an unrecoverable error.
```
defaults write net.culater.SIMBL SIMBLLogLevel -int 2
```