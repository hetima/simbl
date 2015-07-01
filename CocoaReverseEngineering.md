# Armchair Guide To Cocoa Reverse Engineering #

I've authored a few plugins that integrate with existing applications. About as often as I get asked for the source, I get asked how I figured it all out.

It's not rocket science - it's more like archaeology.  Basically, start somewhere, chip away slowly and make a series of progressively more informed decisions.  Obviously that's a pretty gross simplification of the process, but there's no black art - you just need a few hints.  That's basically the purpose of this document - to get you started.  I'm doing this as a wiki page because I envision that there is going to be the need for additional clarification and comments.


# The Basics #

You should know something about Cocoa and Objective-C, since pretty much everything I'm going to talk about relates to it somehow.  You need not be expert by any means, my first use of ProjectBuilder/Cocoa/Objective-C was to hack symbol completion into ProjectBuilder. I learned Cocoa/Objective-C, hacking and reversing pretty much simultaneously.

If you know to use XCode/ProjectBuilder to build a bundle and load it, you are well equipped. Even if you don't, be adventurous and read on.


# Choose Your Target #

So, let's say you want to hack the Terminal.app so you can change the ANSI text colors.  Good choice. Now you've got to figure out how - but without source. Fear not, citizen, there is so much metadata embedded in compiled Objective-C that you can recreate the internal structure of the code quite easily. But you need tools to do so.


# Tools of the Trade #

  * `nm` is included on virtually any Unix-like platform. This dumps out a list of C function names, mangled C++ calls, and Objective-C message names.
  * `strings` is also pretty universal. This dumps out strings from the given binary - which is handier than you'd thing. Oftentimes "secret" preference keys will reveal themselves as you look at the strings within a binary.
  * `gdb` is well equipped to help you on your way. You can run any application from gdb and set breakpoints on Objective-C messages, just as you would with a C function. You can also do some noodling around to explore data structures at runtime. Very powerful, but not the easiest tool to use.
  * [class-dump](http://www.codethecode.com/Projects/class-dump) is your friend. In fact, get to know it like family.  class-dump loads a given chunk of Objective-C code and generates a fairly convincing header file of the classes contained within. This will give you an excellent snapshot of the application and you can learn a lot from the information contained within.
  * [FScript](http://www.fscript.org)/[FScriptAnywhere](http://homepage.mac.com/kenferry/software.html#fsa) is a Smalltalk-like scripting language that integrates nicely with Objective-C and Cocoa.  [FScriptAnywhere](http://homepage.mac.com/kenferry/software.html#fsa) is a SIMBL plugin that loads an [FScript](http://www.fscript.org) interpreter into any Cocoa application. Once the interpreter is loaded, you can explore code objects at runtime - examine values, call methods, etc.  This is obscenely powerful - but you can of course completely corrupt application data as well. Be especially careful with anything that autosaves a database or complex data structure (like iPhoto or Mail) and be sure to backup before you start fiddling.
  * [SIMBL](http://www.culater.net/software/SIMBL/SIMBL.php) is a framework for loading standard bundles into an application when it starts. You basically compile a standard Cocoa bundle (use the default XCode template), but add a couple of special keys to the Info.plist to specify which applications should load your bundle. There is more information and some code fragments further along in the article.


# The Middle Part #

Obviously every application is different. Some are a lot easier to reverse engineer than others. Usually the better engineered the application is, the easier it is to unravel it and make changes.

Figuring out what changes you can make is a little bit challenging and there isn't much to say that can make it easier. You have to chip away and find out what works.  You need to look for methods you can easily override and try and stay away from directly accessing member variables if possible.

My strategy is usually to `classdump` the binary and read the class names, then start searching for relevant method names.  Sometimes using [FScript](http://www.fscript.org) can be a lot easier. If you want to know where a table view gets it's data, you can use the object browser portion of [FScript](http://www.fscript.org) to disclose the view object behind a control. Now you can walk up and down the view hierarchy to find the object you are most interested in. From there you can explore the relevant data sources. Often this takes you "where you want to go" faster.


# Patching Code #

Assuming you've found a suitable method to override (this might seem like a big assumption for now, but eventually you'll find your in) you need to know how to substitute your code for the existing function. There are a couple of "simple" options, but they each have some limitations.

## Posing ##

Class posing is useful technique enabled by the Objective-C runtime. In a nutshell, this allows you to create all future instances of class A as class B, where class B is a subclass of A.
```
	[[B class] poseAsClass:[A class]];
```
I won't go into too much detail - you can read Apple's documentation on class posing or `+ [NSObject poseAsClass:(Class)_class]`.

Posing has a couple of shortcomings.

  1. Only instances created after posing will be created as the subclass. This means that there can be instances of class A lingering around if they were created before you performed the pose. Usually, this doesn't turn out to be a huge problem, but it's good to know up front before you go berserk trying to figure out why your code hasn't been swapped in.
> 2. You cannot add instance variables to the subclass (which mostly defeats the point of the subclass). I'm not 100% sure of the reasoning here, but I'd hazard a guess that it has something to do with some assumptions about the size of an object staying the same.  Fortunately, you can approximate instance variables with the following idiom:
```
static NSMutableDictionary* s_fakeIvars = nil;

+ (void) initialize
{
	s_fakeIvars = [[NSMutableDictionary alloc] init];
}

- (id) init
{
	self = [super init];
	[s_fakeIvars setObject:[NSMutableDictionary dictionary] forKey:[NSNumber numberWithInt:(int)self]];
}

- (void) dealloc
{
	[super dealloc];
	// note: assumes 32-bit pointers
	[s_fakeIvars removeObjectForKey:[NSNumber numberWithInt:(int)self]];
}

```

> 3. If you pose as a class for which you have generated the header (say, with classdump), if the size changes (for example, with a new release of the target application), your plugin will likely cause the application to crash.

> 4. Multiple plugins can pose as the same class - this can get a bit hairy.

## Method Swizzling ##

I forget where I first heard the term "method swizzling" but it is as good a term as any.  Basically, swizzling means that you swap the implementation of a particular method with another. Another way of saying it is "renaming a method."  You are meddling with the internals of the Objective-C runtime, but that's not as evil as it sounds.

Here's a snippet from DuctTape that might be worth 1000 words:

```
/**
 * Renames the selector for a given method.
 * Searches for a method with _oldSelector and reassigned _newSelector to that
 * implementation.
 * @return NO on an error and the methods were not swizzled
 */
BOOL DTRenameSelector(Class _class, SEL _oldSelector, SEL _newSelector)
{
	Method method = nil;

	// First, look for the methods
	method = class_getInstanceMethod(_class, _oldSelector);
	if (method == nil)
		return NO;

	method->method_name = _newSelector;
	return YES;
}
```

This means that the original name for the selector is abandoned and can no longer call the expected code.  This in and of itself might not be that useful, so I frequently use the following idiom:

```
// never implemented, just here to silence a compiler warning
@interface WebInternalImage (PHWebInternalImageSwizzle)
- (void) _webkit_scheduleFrame;
@end

@implementation WebInternalImage (PHWebInternalImage)

+ (void) initialize
{
	DTRenameSelector([self class], @selector(scheduleFrame), @selector (_webkit_scheduleFrame));
	DTRenameSelector([self class], @selector(_ph_scheduleFrame), @selector(scheduleFrame));
}

- (void) _ph_scheduleFrame
{
	// do something crazy...
	...
	// call the "super" method - this method doesn't exist until runtime
	[self _webkit_scheduleFrame];
}

@end
```

This snippet is from the image animation code in WebKit that PithHelmet overrides. Obviously your method and the original method should have the same signature. I usually prepend relevant abbreviations to keep things straight in the code.

Since all method calls in Objective-C are looked up at runtime, anything previously calling `- [WebInternalImage scheduleFrame]` is now routed to your method.  Pretty cool.

Method swizzling has some of the same limitations and workarounds as class posing. You can't add true instance variables, but luckily the technique described above works just fine for swizzling as well. One advantage is that once you have swizzled the method, any object will call your code - whether it's target was created before or after the swizzle.  As I said before, this doesn't turn out to be useful all that often, but when you need it, you'll be glad it works.


# Building Your Patch #

Now you need to get your code into your target application. I'll walk you through a really basic SIMBL plugin.

## Why Use SIMBL? ##

Strictly speaking, this isn't necessary.  You could try and package your plugin as an InputManager, but as you deal with the various shortcomings of that approach, you'd end up writing your own version of SIMBL and wasting a lot of your time resolving really generic problems. This is of course, boring, and that's probably the best reason not to do it.

SIMBL does a few cool things for you:
  * loads Cocoa bundles in a safe way such that your plugin only gets installed in the intended application
  * properly searches the library hierarchy to support user-specific and system-wide plugin installs
  * makes sure only one version of a plugin loads (preferring the user-specified version)
  * allows you to target a specific version of an application to squelch unwanted crashes

Also, I estimate SIMBL is installed on over 100,000 machines (in many different countries), so it has been well tested and most unlikely to create a problem in any application in and of itself.

## Creating A SIMBL Plugin Bundle ##

Creating a SIMBL plugin project is pretty simple, but there are a few things that haven't really ever been properly documented.

  1. Create a new "Cocoa Bundle" project in XCode.
> 2. Create a basic plugin class - say, `MySamplePlugin`. You will use this class as a jumping-off point to setup the rest of your hacks.
> 3. Edit the `Info.plist`
> > a. Set `NSPrincipalClass` to `MySamplePlugin`
> > a. Create a new array key `SIMBLTargetApplications`
> > a. Create a child dictionary of `SIMBLTargetApplications` with the keys `BundleIdentifier`, `MaxBundleVersion` and `MinBundleVersion`.  All of them should have string values.
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

> 4. Build out your plugin class.  You probably want to put most of your initialization code in the `load` class method. This is a special method that SIMBL looks for when it loads a plugin's principal class. `load` gets called at a nice "safe" time when the application has mostly initialized itself and is pretty much ready for interaction with the user. While you can put your initialization code in other methods, I recommend putting it here. Usually I make the plugin class a singleton object, since it's often nice to have a single location to call "home."
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
> 5. Compile. Hopefully you don't get too many errors, but this can sometimes be a little tricky. You might need to set one of these handy flags (LinkerSettings) for hacking in Cocoa (set these under Other Linker Flags)
    * `-undefined suppress`(although, when using a two level namespace, XCode doesn't like this)
    * `-undefined define_a_way`

> These suppress linking warnings when you are building a bundle that you know will get loaded into a memory space where certain functions are assumed to be already defined.  This is instrumental in getting a lot of SIMBL hacks to work.

> 6. Make sure your plugin gets loaded.  Usually I create a user plugins directory and symlink the plugin bundles so I can quickly turn them on and off without affecting other users.
```
mkdir -p ~/Library/Application\ Support/SIMBL/Plugins
cd ~/Library/Application\ Support/SIMBL/Plugins
ln -s ~/MySamplePlugin/build/MySamplePlugin.bundle .
```


# Common Pitfalls #

Things inevitably go wrong and you can blow a lot of time tracking down the problem.  Here are some things likely to bite you:

## Software Updates ##

If you do some class posing or end up having to access member variables directly, and you probably will, be aware that your plugin will probably break when the underlying application gets upgraded. If the `@interface` you use to compile your plugin is out of sync with the actual application, you will get random crashes. This is the first thing I check with any new code release.  You almost always have to resize the objects you extend.

## Symbol Conflicts ##

Things get ugly fast if you have a piece of code with a name conflict. There are 3 common ways for this to happen:
  1. You name a class the exact same thing as another class that already exists in the application's runtime (this includes other plugins that may have been installed at runtime).  I usually pick a prefix of 2 or 3 letters for all the classes for a particular project to avoid naming conflicts.
> 2. You have a category that defines a new method that has the same name as an existing method. Again, I usually us a prefix - especially on categories to Foundation or AppKit objects.
> 3. You define a C function with the same name as an existing C function.

Are you sensing a pattern here? Yeah, prefixes.

# Responsible Patching #

Use version checking to turn off your plugins in applications that just won't work - you don't want to cause other developers/Apple undue stress by breaking every application upgrade. Trust me - I learned the hard way. I'm sure there is a blacklist with my name on it near the Safari developers.


# What Now? #

Go out and play. You just need to tinker a bit to get the hang of it. If you need help, drop a comment on the wiki or send me some email.