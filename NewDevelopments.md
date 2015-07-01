# 2011-01-27 SIMBL 0.9.9 #
  * fix **exceedingly rare** bug causing the installer to erroneously report failure
  * fix **rare** bug causing the agent to fail ungracefully on malformed bundles


# 2010-10-21 SIMBL 0.9.8 #
  * fixed for Leopard (10.5.x)
  * many small installer fixes for rare issues
  * fixed uinstaller

# 2009-10-04 SIMBL 0.9.7 #
  * blacklist for applications that cause the agent to crash
  * adjustable logging for debugging
  * quieter logging by default

# 2009-09-16 SIMBL 0.9.6 #

  * fixed installer to repair even worse permissions
  * injection mechanism is now much more efficient - only affect applications that have plugins

# 2009-09-10 SIMBL 0.9.5 #

The installer for this version will repair broken permissions that could prevent the SIMBL Agent from launching.  Additionally, this is the first version to ship with an uninstaller as well.

# 2009-09-08 SIMBL 0.9.4 #

This version fixes a bug where certain applications (most notably Adobe apps) would result in a dialog asking where an application could be found. I also recompiled with a much leaner set of linked libraries which should speed up the launch process a bit. My goal is to add the absolute minimum overhead.

# 2009-09-07 SIMBL 0.9.3 #

This version should be compatible with garbage collected applications. It also fixes the installer to remove conflicting versions of the old SIMBL InputManager.

# 2009-09-06 SIMBL for Leopards #

SIMBL-0.9.2b seems to work on both Leopard and Snow Leopard. Existing bundles should load fine in 32-bit mode as-is, no modifications. 64-bit bundles should work as well - I've tested this with a trivial plugin inside Safari.

This should be considered beta quality. In 5 years, I had one actual bug reported against SIMBL as an InputManager, but this is very different code with a very different mechanism.  I have some ideas on how to increase reliability and minimize system impact, but I guessed a quick release would prevent people from reinventing too many wheels.

# 2009-09-03 SIMBL, Snow Leopard and the 64-bit Conundrum #

SIMBL apparently works under some very specific circumstances on Snow Leopard.
  1. the application you are running is 32-bit
  1. the permissions and ownership on the /Library/InputManagers folder is exactly correct

There may even be more restrictions I'm not yet entirely aware of.

Now, you can **force** some applications to run as 32-bit, but unfortunately this not a general solution.

To address this, I've rebuilt the code injection mechanism in a two-part system. There is a small chunk of code that implements an OSAX scripting extension. There is an even smaller chunk of code that watches for application launches and triggers the injection of SIMBL when the application has finished launching.  This is installed as an agent and runs in the background.

This seems to work fine on my Leopard PPC machine. I'm hoping I can get the remaining issues ironed out and tested on Snow Leopard in 64-bit mode tonight.

In the meantime, I will cobble together a bit of an installer. This will require some testing as well, but I'm fairly confident something will be available in the next couple of days.

There are a few solutions that I discarded.

Hacking a value for DYLD\_INSERT\_LIBRARIES into ~/.MacOSX/environment.plist initially looked promising. Despite obvious installer hassle (each installation only works with the current user, or must modify all users currently on the system) what really removed it from consideration was the side effects.

Using DYLD\_INSERT\_LIBRARIES requires DYLD\_FORCE\_FLAT\_NAMESPACE. This could break a lot of things. However, even worse is that this mechanism is **much less** forgiving of deleting and removing the SIMBL framework itself. If the SIMBL library is removed or becomes broken, the potential is there for all applications to fail launching. This is obviously horrible, so I abandoned this pretty quickly.