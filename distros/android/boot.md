Boot SF Trace

 - at first, only BootAnimation
 - then, in addition to BootAnimation, there are invisible Display Root and Display Overlays
   - Display Root also has many invisible children
     - mBelowAppWindowsContainers
     - com.android.server.wm.DisplayContent$TaskStackContainers@81d9ce7
       - animationLayer
       - boostedAnimationLayer
       - homeAnimationLayer
       - splitScreenDividerAnchor
       - Stack=0
         - animation background staciId=0
     - mAboveAppWindowsContainers
     - mImeWindowsContainers
 - then there is invisible WallpaperWindowToken under mBelowAppWindowsContainers
 - then there is visible com.android.settings/com.android.settings.FallbackHome under Task=1 under Stack=0
   - only for a few frames
 - then there are StatusBar, NavigationBar, ScreenDecorOverlay, ScreenDecorOverlayBottom
 - then BootAnimation disappeared
 - then StatusBar becomes full sreen and NavigationBar disappeared
 - then wallpaper is visible

Lockscreen is
 - full screen status bar

First Boot

 - BootAnimation appears
 - setupwizard appears
 - StatusBar appears
 - ScreenDecorOverlay appears
 - BootAnimation disappears
