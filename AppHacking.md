Here’s some example code to load the bundle assuming it’s simply in the Resources/ directory of your application’s main bundle:
```
    NSString* pathToBundle = [[NSBundle mainBundle] pathForResource:@"MyExtraClasses"
                                                             ofType:@"bundle"];
    NSBundle* bundle = [NSBundle bundleWithPath:pathToBundle];
    
    Class myClass = [bundle classNamed:@"MyClass"];
    NSAssert(myClass != nil, @"Couldn't load MyClass");
    [myClass initialize];
```
You will also have to end up specifying using the -weak parameters to the linker (such as -weak\_framework or -weak-l) for your main application if you’re still using normal methods from that class. That’s about it though, and conceptually this technique is quite robust.