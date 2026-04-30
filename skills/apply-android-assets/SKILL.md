---
name: "apply-android-assets"
description: "Generates and applies Android icon.svg and splash.svg using @capacitor/assets. Invoke when user wants to update the app icon or splash screen for Android."
---

# Apply Android Assets

This skill automates the process of generating and correctly applying Android icons and splash screens from SVG files in Capacitor projects.

## When to use
Invoke this skill when:
- The user provides or updates `icon.svg` or `splash.svg` and wants to apply them.
- The user complains about stretched, blank, or incorrect splash screens on Android.
- The user asks to generate Android assets.

## Prerequisites
- The source files `resources/icon.svg` and `resources/splash.svg` must exist.
- The `@capacitor/assets` package should be available (can be run via `npx @capacitor/assets`).

## Instructions

1. **Prepare Dark Mode Assets**:
   If the project doesn't have a specific `splash-dark.svg`, the assets generator will default to an icon on a black background for dark mode, which will override the splash background. Always ensure a dark variant exists:
   ```bash
   # In PowerShell/bash
   cp resources/splash.svg resources/splash-dark.svg
   ```

2. **Generate Assets**:
   Run the following command to generate the Android assets from the `resources` folder:
   ```bash
   npx @capacitor/assets generate --android
   ```
   *Note: If the package is not installed, this will download and run it.*

3. **Fix Splash Screen Background Configuration**:
   Capacitor's default splash screen generation often results in a stretched image or incorrect background. You must manually configure the background layer.

   - **Step 3.1**: Check `android/app/src/main/res/values/styles.xml` (and `values-night/styles.xml` if it exists).
   - Ensure the `AppTheme.NoActionBarLaunch` style points to a custom background drawable, rather than just `@drawable/splash`:
     ```xml
     <style name="AppTheme.NoActionBarLaunch" parent="AppTheme.NoActionBar">
         <item name="android:windowBackground">@drawable/splash_background</item>
     </style>
     ```

   - **Step 3.2**: Create or update `android/app/src/main/res/drawable/splash_background.xml` to compose the background color and the centered splash image.
     
     **Example with a Solid Background:**
     ```xml
     <?xml version="1.0" encoding="utf-8"?>
     <layer-list xmlns:android="http://schemas.android.com/apk/res/android">
         <!-- Background Color Layer -->
         <item>
             <shape>
                 <solid android:color="#3880ff" /> <!-- Replace with the actual brand color from capacitor.config.ts or colors.xml -->
             </shape>
         </item>
         <!-- Centered Splash Image Layer -->
         <item>
             <bitmap
                 android:src="@drawable/splash"
                 android:gravity="center" />
         </item>
     </layer-list>
     ```

     **Example with a Gradient Background:**
     ```xml
     <?xml version="1.0" encoding="utf-8"?>
     <layer-list xmlns:android="http://schemas.android.com/apk/res/android">
         <!-- Background Gradient Layer -->
         <item>
             <shape>
                 <gradient 
                     android:startColor="#3880ff" 
                     android:endColor="#2dd36f" 
                     android:angle="315" />
             </shape>
         </item>
         <!-- Centered Splash Image Layer -->
         <item>
             <bitmap
                 android:src="@drawable/splash"
                 android:gravity="center" />
         </item>
     </layer-list>
     ```
   - **Color reference**: Use the `backgroundColor` defined in `capacitor.config.ts` (`plugins.SplashScreen.backgroundColor`) or the project's primary color.

4. **Verify App Configuration & Hide Splash Screen Logic**:
   - Ensure `capacitor.config.ts` has the correct splash screen plugin settings (e.g., `splashFullScreen: true`, `splashImmersive: true`).
   - **IMPORTANT**: Set `launchAutoHide: false` in `capacitor.config.ts` so the splash screen isn't hidden prematurely.
   - Hide the splash screen manually when the main App component is mounted. For example, in React (`App.tsx`):
     ```typescript
     import { SplashScreen } from '@capacitor/splash-screen';
     import { useEffect } from 'react';

     // Inside your main App component:
     useEffect(() => {
       // Hide splash screen once the app component is mounted
       const hideSplash = async () => {
         try {
           await SplashScreen.hide();
         } catch (err) {
           console.warn('Error hiding splash screen', err);
         }
       };
       hideSplash();
     }, []);
     ```

5. **Testing & Committing**:
   - Run tests if required by the workspace rules.
   - If committing changes, remember to use `--no-pager` for git commands (e.g., `git --no-pager diff`) as per the workspace `RULES.md`.
