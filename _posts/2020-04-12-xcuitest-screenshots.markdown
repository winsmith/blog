---
layout: post
title:  "Creating automated Screenshots using XCUITest"
subtitle: "Look ma, no fastlane!"
date:   2020-04-14 00:00:00 +0000
categories: english ios
---

<p class="lead">If you develop an iOS app, you know this: <strong>Creating screenshots is a lot of work.</strong> You need updated screenshots for all your device types in all your languages, and manually creating all those takes a long time. </p>

Lots of people are using [fastlane](https://fastlane.tools) to create their screenshots, but for a variety of reasons, I decided against it. I had tried it out before, but it doesn't really fit into my development workflow, and it feels like too much for just taking screenshots.

Fastlane uses XCUITest to take screenshots anyway, so I decided to try out if I could **recreate the functionality of `fastlane snapshot`** without using Ruby, without installing any gems, and without having to buy into the fastlane ecosystem.

<img class="img-fluid" src="/assets/libi-ui-tests.gif" alt="A screen recording of screenshots being created">

My goal: Write an Xcode UI Test that takes screenshots, then extract and collect those screenshots for all languages and devices into a folder on my Desktop. I'm going to use my own app, <a href="https://apps.apple.com/us/app/libi/id1475819970?uo=4">Libi</a>, as an example.

After some fiddling, here are my results.

## Overview over the Process

Here are the steps we need to take:

1. Run Xcode UI Tests that take screenshots
1. Run the Tests from the command line
1. Extract those screenshots from the test logs
1. Run those steps for each combination of device and language.

Below are my solutions to all 4 steps, plus some addenda that make things easier. If you're stuck, read through to the bottom to see if a solution to your problen is already there. 

## First Step: Run Xcode UI Tests that Take Screenshots

Just a quick sneak preview, here is how things should look like at the end:

<img class="img-fluid" src="/assets/all_code.png" alt="An Xcode screenshot displaying the finished written test">

### Setting Things up

Even if you already have an UI Testing Target, you should probably create a new one for your screenshots. If you do not have an UI Testing Target, welcome to the world of UI testing! 

1. In Xcode, go to `File > New > Target...` and select “UI testing bundle”.
1. Call the Bundle something like "YourAppScreenshots" and click "Finish".
1. Edit your app’s scheme to run your UI tests when testing: Go to `Product > Scheme > Edit Scheme...` click on “Test” in the sidebar, and enable your new bundle. (Newer Xcode versions might already have it enabled, so no need to do anything in this case)

<div class="alert alert-info" role="alert">
  <strong>Note:</strong> I wrote this article against Xcode 11.4 and Swift 5. If you use different versions, your experience may differ. 
</div>

### Create a Test

Add a new property to your test class to keep track of the `XCUIApplication`:

```swift
var app: XCUIApplication!
```

then initialize the `app` property in your setup method:

```swift
override func setUpWithError() throws {
    continueAfterFailure = false
    app = XCUIApplication()

    // We send a command line argument to our app,
    // to enable it to reset its state
    app.launchArguments.append("--libiScreenshots")
}
```

Now you can create your first test. I put the whole screenshot flow into one test, but might split individual flows off into their own test functions later:

```swift
func testMakeScreenshots() {
    app.launch()
    
    // Overview
    takeScreenshot(named: "Overview")

    // Insights
    app.tabBars.buttons["Deep Insights"].tap()
    takeScreenshot(named: "Insights")
}
```

This test uses a function I wrote, `takeScreenshot(named:)`, which extracts the correct way of taking screenshots in our situation. See below for the function code.

<div class="alert alert-info" role="alert">
  <strong>Note:</strong> If you don't want to write your UI test yourself, or you want an easy starting point, you can try <strong>recording</strong> your steps instead. To do that, place the cursor inside your <code>testMakeScreenshot</code> function and click the little 🔴&nbsp;record button in the bottom toolbar. This will launch your app in the simulator and then record your taps there as code inside the test function. Click the record button again to stop.
</div>

<div class="alert alert-info" role="alert">
  <strong>Another Note:</strong> The Xcode UI testing framework is pretty dependent on <strong><a href="https://developer.apple.com/documentation/uikit/uiaccessibilityidentification/1623132-accessibilityidentifier">accessibility identifiers</a></strong>. If you don't explicitly set those, UIKit will use the text values of things like buttons or labels as accessibility identifiers, but since these are localized, your tests might not work after switching the scheme language. My recommendation is to set the <code>accessibilityIdentifier</code> property of things you need to access in your tests real quick.
</div>

### Take Screenshots

Since we want to identify our screenshots, we name them. And since we want to keep them until after the end of the test is run, we'll also have to mark them as "keep always".

```swift
func takeScreenshot(named name: String) {
    // Take the screenshot
    let fullScreenshot = XCUIScreen.main.screenshot()
    
    // Create a new attachment to save our screenshot
    // and give it a name consisting of the "named"
    // parameter and the device name, so we can find
    // it later.
    let screenshotAttachment = XCTAttachment(
        uniformTypeIdentifier: "public.png", 
        name: "Screenshot-\(UIDevice.current.name)-\(name).png",
        payload: fullScreenshot.pngRepresentation, 
        userInfo: nil)
        
    // Usually Xcode will delete attachments after 
    // the test has run; we don't want that!
    screenshotAttachment.lifetime = .keepAlways
    
    // Add the attachment to the test log, 
    // so we can retrieve it later
    add(screenshotAttachment)
}
```

### Aside: Command Line Arguments

Earlier, we set a command line argument. Here's a quick example code on how to retrieve that in your `AppDelegate.swift`:

```swift
// AppDelegate.swift

func application(_ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?
    ) -> Bool {

    if CommandLine.arguments.contains("--libiScreenshots") {
      // We are in testing mode, make arrangements
      configStore = ScreenshotsConfigStore()
      configStore.dataPointsManager.makeExampleData()
    }
    
    // ...
```

### Check if the Screenshots are There

After this step, you can already see the screen shots in the test logs. Make sure that is the case.

1. In Xcode, run your new test (e.g. by pressing <kbd>⌘</kbd><kbd>U</kbd>)
1. Open the Report Navigator and wait for your test for finish
1. Click the topmost test, which is the one you just started
1. Click the Quicklook icon next to one of the test's attachment to see one of your screenshots

<img class="img-fluid" src="/assets/test_report.png" alt="A finished test report in Xcode">

Yay, we have screenshots. That's the hardest part done. But we're not there yet. We need extraction, and automation.

<div class="alert alert-info" role="alert">
  <strong>A Note on Idling:</strong> If you have a lot of <code>Waiting for application xyz to idle</code> messages in your logs, or your test workflows are generally flaky, this might be because of animations that are running. If you set up a command line flag earlier, you can use this to tell your view controllers to disable animations by calling <code>UIView.setAnimationsEnabled(bool)</code>. Here's an example where I use my implementation of a Config Store: 

<p>
<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">override</span> <span class="kd">func</span> <span class="nf">viewDidLoad</span><span class="p">()</span> <span class="p">{</span>
    <span class="k">super</span><span class="o">.</span><span class="nf">viewDidLoad</span><span class="p">()</span>
    
    <span class="c1">// configStore.shouldAnimateUI is true</span>
    <span class="c1">// except when we are running UI Tests</span>
    <span class="kt">UIView</span><span class="o">.</span><span class="nf">setAnimationsEnabled</span><span class="p">(</span><span class="n">configStore</span><span class="o">.</span><span class="n">shouldAnimateUI</span><span class="p">)</span>
    
    <span class="c1">// ...</span>
<span class="p">}</span>
</code></pre></div></div>
</p>

</div>

## Second Step: Run Tests from the Command Line

If we want to automate things, it's a good idea to run them from the command line. Go to your terminal, navigate to the directory that contains your `.xcodeproj` file and run the following command (replace scheme name and project file name)

```bash
$ xcodebuild -testLanguage de -scheme Libi -project ./Libi.xcodeproj -derivedDataPath '/tmp/LibiDerivedData/' -destination "platform=iOS Simulator,name=iPhone 11 Pro Max" build test
```

If the Simulator app is currently running on your system, you'll be able to watch live as the test taps through your app. Otherwise the tests will run in headless mode.

After the tests are done, you'll find a test result `xcresult` file in the derived data Path, `/tmp/LibiDerivedData/Logs/Test/`. It is a package, and it contains all the data we already saw in Xcode.

However, the data is not usable right now. There are no PNG files here, which is disappointing. 

<div class="alert alert-info" role="alert">
  <strong>Note:</strong> In fact, if you watch the .xcresult package contents while the test is actually running, you can even see your screenshots being saved in there as PNGs. However, they then get packaged into the .xcresult format, which is a bit annoying. I wish we could just tell Xcode to leave them out in the open.
</div>

## Third Step: Extract Screenshots from the Test Logs

The `.xcresult` package format is pretty complicated and I didn't really want to parse it myself; luckily I didn't have to. There's a tool called [`xcparse`](https://github.com/ChargePoint/xcparse) which does exactly that. Install it via homebrew:

```bash
$ brew install chargepoint/xcparse/xcparse
```

After installing `xcparse`, you can use its `screenshots` command to extract all screenshots from a given `xcresult` into a folder like so (replace folder names as necessary)

```bash
$ xcparse screenshots /tmp/LibiDerivedData/Logs/Test/Run-Libi-2020.04.13_13-06-03-+0200.xcresult "~/Desktop/LibiScreenshots/"
```

## Fourth Step: Run for each Combination of Device and Language.

And we are basically done. All we have to do now is run the `xcodebuild` and `xcparse` commands for each combination of language and simulator and collect the screenshots in a central location. Let's dust off these bash scripting skills:

```bash
#!/bin/bash

# The Xcode project to create screenshots for
projectName="./Libi.xcodeproj"

# The scheme to run tests for
schemeName="Libi"


# All the simulators we want to screenshot
# Copy/Paste new names from Xcode's
# "Devices and Simulators" window
# or from `xcrun simctl list`.
simulators=(
    "iPhone 8"
    "iPhone 11 Pro"
    "iPhone 11 Pro Max"
    "iPad Pro (12.9-inch) (3rd generation)"
    "iPad Pro (9.7-inch)"
)

# All the languages we want to screenshot (ISO 3166-1 codes)
languages=(
    "en"
    "de"
    "fr"
)

# All the appearances we want to screenshot
# (options are "light" and "dark")
appearances=(
    "light"
    "dark"
)

# Save final screenshots into this folder (it will be created)
targetFolder="/Users/breakthesystem/Desktop/LibiScreenshots"


## No need to edit anything beyond this point


for simulator in "${simulators[@]}"
do
    for language in "${languages[@]}"
    do
        for appearance in "${appearances[@]}"
        do
            rm -rf /tmp/LibiDerivedData/Logs/Test
            echo "📲  Building and Running for $simulator in $language"

            # Boot up the new simulator and set it to 
            # the correct appearance
            xcrun simctl boot "$simulator"
            xcrun simctl ui "$simulator" appearance $appearance

            # Build and Test
            xcodebuild -testLanguage $language -scheme $schemeName -project $projectName -derivedDataPath '/tmp/LibiDerivedData/' -destination "platform=iOS Simulator,name=$simulator" build test
            echo "🖼  Collecting Results..."
            mkdir -p "$targetFolder/$simulator/$language/$appearance"
            find /tmp/LibiDerivedData/Logs/Test -maxdepth 1 -type d -exec xcparse screenshots {} "$targetFolder/$simulator/$language/$appearance" \;
        done
    done

    echo "✅  Done"
done

```

I called this file `takeScreenshots.sh` and added it to the Xcode project for easier editing, inside the `LibiScreenshots` folder. In the repository root folder I can call it like so:

```bash
$ bash LibiScreenshots/takeScreenshots.sh
```

and it should run for a while and take a huge swath of screenshots!

<img class="img-fluid" src="/assets/test_in_action.png" alt="The shell script running and creating screenshots">

Thanks for reading! Feel free to mention me on [Twitter](https://twitter.com/breakthesystem/) if you have questions or comments.


<div class="card mb-3" style="max-width: 540px;">
  <div class="row no-gutters">
    <div class="col-md-4">
      <img src="/assets/libi_icon.png" class="card-img" alt="Libi-Icon">
    </div>
    <div class="col-md-8">
      <div class="card-body">
        <p class="card-text">And check out <a href="https://apps.apple.com/us/app/libi/id1475819970?uo=4">Libi</a>, my free app for tracking your mood, energy level, and libido. It's a labor of love, super privacy focused, and aims to improve your life and mental health with sophisticated analyses and awareness of where your head is at. Thanks 💙</p>
      </div>
    </div>
  </div>
</div>




----

## Addenda

These are things I intend to write about in one or more follow up articles if people are interested

- Addendum Four: Creating Initial Data
- Launch Arguments and Resetting States
- Accessibility Identifiers
- Disabling Animations
- Sleeping, Swiping, and generally controlling UI Tests
- Dark Mode Screenshots (https://stackoverflow.com/questions/59447982/how-to-set-dark-mode-in-xcuiapplication-in-swift-uitests)

----

## Sources

- [Apple WWDC: Testing in Xcode](https://developer.apple.com/videos/play/wwdc2019/413/)
- [Paul Hudson: Xcode UI Testing Cheat Sheet](https://www.hackingwithswift.com/articles/148/xcode-ui-testing-cheat-sheet)
- [Derik Ramirez: Understanding XCUITest screenshots and how to access them](https://rderik.com/blog/understanding-xcuitest-screenshots-and-how-to-access-them/)
- [Swift by Sundell: Getting started with Xcode UI testing in Swift](https://www.swiftbysundell.com/articles/getting-started-with-xcode-ui-testing-in-swift/)
- [ChargePoint: xcparse](https://github.com/ChargePoint/xcparse)
- [NSHipster: simctl](https://nshipster.com/simctl/)
- [Jeff Kelley: Fixing Simulator Status Bars for App Store Screenshots With Xcode 11 and iOS 13](https://www.detroitlabs.com/blog/2019/09/19/fixing-simulator-status-bars-for-app-store-screenshots-with-xcode-11-and-ios-13/)
