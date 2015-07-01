## TODO ##

  * make sure bundle loading only happens in the main thread.
  * pursue better user interface that allows plugins to be turned on and off more simply
    * could be done as a pref pane.
  * create a particular extension type that would allow SIMBL bundles to be automatically installed without using the main Installer.

## FIXED ##

  * clean up logging and make it adjustable via a preference
  * only send the inject event for applications that already match the list of loadable plugins
    * SIMBLAgent can watch the plugins directory. When new plugins are added, keep a hash of all the bundle identifiers that will result in actually loading a plugin. This keeps us from running a costly inject at every application launch.
  * better event targeting for the SIMBL Agent
    * send events based on pid or psn rather than just the name of the application