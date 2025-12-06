================================================
FILE: README.md
================================================
# BroadcastWriter
Simple wrapper for `AVFoundation` `AVAssetWriter`; for writing asset during Broadcast Extension activity

Usage:

1. Add `Broadcast Extension` target
2. Use Swift Package Manager to add dependency to your `Broadcast Extension` target
3. Add `App Groups` capability into main and `Broadcast Extension` targets
4. Call appropriate methods in your `RPBroadcastSampleHandler` instance in `Broadcast Extension` target (see Example project in repo)
5. When `BroadcastWriter` finished, don't forget to move Broadcast Writer output file to your AppGroup container directory, to have access from main target, see [Example](https://github.com/romiroma/BroadcastWriter/blob/75a0b43bd0d17521a2226aed77d43654b921db01/BroadcastUpload/SampleHandler.swift#L73-L113) 
6. Pay Attention to use your own `App Group` identifiers when using `Example` project

![IMG_AD267582BFF1-1](https://user-images.githubusercontent.com/25149401/109422414-182c6880-79e4-11eb-8a47-422e71345191.jpeg)



================================================
FILE: LICENSE
================================================
MIT License

Copyright (c) 2021 Roman Andrykevych

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.



================================================
FILE: Package.swift
================================================
// swift-tools-version:5.3

import PackageDescription

let package = Package(
    name: "BroadcastWriter",
    platforms: [.iOS(.v13)],
    products: [
        .library(
            name: "BroadcastWriter",
            targets: ["BroadcastWriter"]),
    ],
    dependencies: [
    ],
    targets: [
        .target(
            name: "BroadcastWriter",
            dependencies: []),
        .testTarget(
            name: "BroadcastWriterTests",
            dependencies: ["BroadcastWriter"]),
    ]
)



================================================
FILE: BroadcastUpload/BroadcastUpload.entitlements
================================================
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.security.application-groups</key>
	<array>
		<string>group.com.andrykevych.Example</string>
	</array>
</dict>
</plist>



================================================
FILE: BroadcastUpload/Info.plist
================================================
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>$(DEVELOPMENT_LANGUAGE)</string>
	<key>CFBundleDisplayName</key>
	<string>BroadcastUpload</string>
	<key>CFBundleExecutable</key>
	<string>$(EXECUTABLE_NAME)</string>
	<key>CFBundleIdentifier</key>
	<string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
	<key>CFBundleInfoDictionaryVersion</key>
	<string>6.0</string>
	<key>CFBundleName</key>
	<string>$(PRODUCT_NAME)</string>
	<key>CFBundlePackageType</key>
	<string>$(PRODUCT_BUNDLE_PACKAGE_TYPE)</string>
	<key>CFBundleShortVersionString</key>
	<string>1.0</string>
	<key>CFBundleVersion</key>
	<string>1</string>
	<key>NSExtension</key>
	<dict>
		<key>NSExtensionPointIdentifier</key>
		<string>com.apple.broadcast-services-upload</string>
		<key>NSExtensionPrincipalClass</key>
		<string>$(PRODUCT_MODULE_NAME).SampleHandler</string>
		<key>RPBroadcastProcessMode</key>
		<string>RPBroadcastProcessModeSampleBuffer</string>
	</dict>
</dict>
</plist>



================================================
FILE: BroadcastUpload/SampleHandler.swift
================================================
//
//  SampleHandler.swift
//  BroadcastUpload
//
//  Created by Roman on 28.02.2021.
//

import BroadcastWriter
import ReplayKit
import UserNotifications

class SampleHandler: RPBroadcastSampleHandler {

    private var writer: BroadcastWriter?
    private let fileManager: FileManager = .default
    private let notificationCenter = UNUserNotificationCenter.current()
    private let nodeURL: URL

    override init() {
        nodeURL = fileManager.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
            .appendingPathExtension(for: .mpeg4Movie)

        fileManager.removeFileIfExists(url: nodeURL)

        super.init()
    }

    override func broadcastStarted(withSetupInfo setupInfo: [String : NSObject]?) {
        let screen: UIScreen = .main
        do {
            writer = try .init(
                outputURL: nodeURL,
                screenSize: screen.bounds.size,
                screenScale: screen.scale
            )
        } catch {
            assertionFailure(error.localizedDescription)
            finishBroadcastWithError(error)
            return
        }
        do {
            try writer?.start()
        } catch {
            finishBroadcastWithError(error)
        }
    }

    override func processSampleBuffer(_ sampleBuffer: CMSampleBuffer, with sampleBufferType: RPSampleBufferType) {
        guard let writer = writer else {
            debugPrint("processSampleBuffer: Writer is nil")
            return
        }

        do {
            let captured = try writer.processSampleBuffer(sampleBuffer, with: sampleBufferType)
            debugPrint("processSampleBuffer captured", captured)
        } catch {
            debugPrint("processSampleBuffer error:", error.localizedDescription)
        }
    }

    override func broadcastPaused() {
        debugPrint("=== paused")
        writer?.pause()
    }

    override func broadcastResumed() {
        debugPrint("=== resumed")
        writer?.resume()
    }

    override func broadcastFinished() {
        guard let writer = writer else {
            return
        }

        let outputURL: URL
        do {
            outputURL = try writer.finish()
        } catch {
            debugPrint("writer failure", error)
            return
        }

        guard let containerURL = fileManager.containerURL(
                    forSecurityApplicationGroupIdentifier: "group.com.andrykevych.Example"
        )?.appendingPathComponent("Library/Documents/") else {
            fatalError("no container directory")
        }
        do {
            try fileManager.createDirectory(
                at: containerURL,
                withIntermediateDirectories: true,
                attributes: nil
            )
        } catch {
            debugPrint("error creating", containerURL, error)
        }

        let destination = containerURL.appendingPathComponent(outputURL.lastPathComponent)
        do {
            debugPrint("Moving", outputURL, "to:", destination)
            try self.fileManager.moveItem(
                at: outputURL,
                to: destination
            )
        } catch {
            debugPrint("ERROR", error)
        }

        debugPrint("FINISHED")
    }

    private func scheduleNotification() {
        print("scheduleNotification")
        let content: UNMutableNotificationContent = .init()
        content.title = "broadcastStarted"
        content.subtitle = Date().description

        let trigger: UNNotificationTrigger = UNTimeIntervalNotificationTrigger.init(timeInterval: 5, repeats: false)
        let notificationRequest: UNNotificationRequest = .init(
            identifier: "com.andrykevych.Some.broadcastStarted.notification",
            content: content,
            trigger: trigger
        )
        notificationCenter.add(notificationRequest) { (error) in
            print("add", notificationRequest, "with ", error?.localizedDescription ?? "no error")
        }
    }
}

extension FileManager {

    func removeFileIfExists(url: URL) {
        guard fileExists(atPath: url.path) else { return }
        do {
            try removeItem(at: url)
        } catch {
            print("error removing item \(url)", error)
        }
    }
}



================================================
FILE: BroadcastUploadSetupUI/BroadcastSetupViewController.swift
================================================
//
//  BroadcastSetupViewController.swift
//  BroadcastUploadSetupUI
//
//  Created by Roman on 28.02.2021.
//

import ReplayKit

class BroadcastSetupViewController: UIViewController {

    // Call this method when the user has finished interacting with the view controller and a broadcast stream can start
    func userDidFinishSetup() {
        // URL of the resource where broadcast can be viewed that will be returned to the application
        let broadcastURL = URL(string:"http://apple.com/broadcast/streamID")
        
        // Dictionary with setup information that will be provided to broadcast extension when broadcast is started
        let setupInfo: [String : NSCoding & NSObjectProtocol] = ["broadcastName": "example" as NSCoding & NSObjectProtocol]
        
        // Tell ReplayKit that the extension is finished setting up and can begin broadcasting
        self.extensionContext?.completeRequest(withBroadcast: broadcastURL!, setupInfo: setupInfo)
    }
    
    func userDidCancelSetup() {
        let error = NSError(domain: "YouAppDomain", code: -1, userInfo: nil)
        // Tell ReplayKit that the extension was cancelled by the user
        self.extensionContext?.cancelRequest(withError: error)
    }
}



================================================
FILE: BroadcastUploadSetupUI/Info.plist
================================================
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>$(DEVELOPMENT_LANGUAGE)</string>
	<key>CFBundleDisplayName</key>
	<string>BroadcastUploadSetupUI</string>
	<key>CFBundleExecutable</key>
	<string>$(EXECUTABLE_NAME)</string>
	<key>CFBundleIdentifier</key>
	<string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
	<key>CFBundleInfoDictionaryVersion</key>
	<string>6.0</string>
	<key>CFBundleName</key>
	<string>$(PRODUCT_NAME)</string>
	<key>CFBundlePackageType</key>
	<string>$(PRODUCT_BUNDLE_PACKAGE_TYPE)</string>
	<key>CFBundleShortVersionString</key>
	<string>1.0</string>
	<key>CFBundleVersion</key>
	<string>1</string>
	<key>NSExtension</key>
	<dict>
		<key>NSExtensionAttributes</key>
		<dict>
			<key>NSExtensionActivationRule</key>
			<dict>
				<key>NSExtensionActivationSupportsReplayKitStreaming</key>
				<true/>
			</dict>
		</dict>
		<key>NSExtensionPointIdentifier</key>
		<string>com.apple.broadcast-services-setupui</string>
		<key>NSExtensionPrincipalClass</key>
		<string>$(PRODUCT_MODULE_NAME).BroadcastSetupViewController</string>
	</dict>
</dict>
</plist>



================================================
FILE: Example/AppDelegate.swift
================================================
//
//  AppDelegate.swift
//  Example
//
//  Created by Roman on 28.02.2021.
//

import UIKit

@main
class AppDelegate: UIResponder, UIApplicationDelegate {



    func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        // Override point for customization after application launch.
        return true
    }

    // MARK: UISceneSession Lifecycle

    func application(_ application: UIApplication, configurationForConnecting connectingSceneSession: UISceneSession, options: UIScene.ConnectionOptions) -> UISceneConfiguration {
        // Called when a new scene session is being created.
        // Use this method to select a configuration to create the new scene with.
        return UISceneConfiguration(name: "Default Configuration", sessionRole: connectingSceneSession.role)
    }

    func application(_ application: UIApplication, didDiscardSceneSessions sceneSessions: Set<UISceneSession>) {
        // Called when the user discards a scene session.
        // If any sessions were discarded while the application was not running, this will be called shortly after application:didFinishLaunchingWithOptions.
        // Use this method to release any resources that were specific to the discarded scenes, as they will not return.
    }


}




================================================
FILE: Example/Example.entitlements
================================================
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>com.apple.security.application-groups</key>
	<array>
		<string>group.com.andrykevych.Example</string>
	</array>
</dict>
</plist>



================================================
FILE: Example/Info.plist
================================================
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>CFBundleDevelopmentRegion</key>
	<string>$(DEVELOPMENT_LANGUAGE)</string>
	<key>CFBundleDisplayName</key>
	<string>Example</string>
	<key>CFBundleExecutable</key>
	<string>$(EXECUTABLE_NAME)</string>
	<key>CFBundleIdentifier</key>
	<string>$(PRODUCT_BUNDLE_IDENTIFIER)</string>
	<key>CFBundleInfoDictionaryVersion</key>
	<string>6.0</string>
	<key>CFBundleName</key>
	<string>$(PRODUCT_NAME)</string>
	<key>CFBundlePackageType</key>
	<string>$(PRODUCT_BUNDLE_PACKAGE_TYPE)</string>
	<key>CFBundleShortVersionString</key>
	<string>1.0</string>
	<key>CFBundleVersion</key>
	<string>1</string>
	<key>LSRequiresIPhoneOS</key>
	<true/>
	<key>UIApplicationSceneManifest</key>
	<dict>
		<key>UIApplicationSupportsMultipleScenes</key>
		<false/>
		<key>UISceneConfigurations</key>
		<dict>
			<key>UIWindowSceneSessionRoleApplication</key>
			<array>
				<dict>
					<key>UISceneConfigurationName</key>
					<string>Default Configuration</string>
					<key>UISceneDelegateClassName</key>
					<string>$(PRODUCT_MODULE_NAME).SceneDelegate</string>
					<key>UISceneStoryboardFile</key>
					<string>Main</string>
				</dict>
			</array>
		</dict>
	</dict>
	<key>UIApplicationSupportsIndirectInputEvents</key>
	<true/>
	<key>UILaunchStoryboardName</key>
	<string>LaunchScreen</string>
	<key>UIMainStoryboardFile</key>
	<string>Main</string>
	<key>UIRequiredDeviceCapabilities</key>
	<array>
		<string>armv7</string>
	</array>
	<key>UISupportedInterfaceOrientations</key>
	<array>
		<string>UIInterfaceOrientationPortrait</string>
		<string>UIInterfaceOrientationLandscapeLeft</string>
		<string>UIInterfaceOrientationLandscapeRight</string>
	</array>
	<key>UISupportedInterfaceOrientations~ipad</key>
	<array>
		<string>UIInterfaceOrientationPortrait</string>
		<string>UIInterfaceOrientationPortraitUpsideDown</string>
		<string>UIInterfaceOrientationLandscapeLeft</string>
		<string>UIInterfaceOrientationLandscapeRight</string>
	</array>
</dict>
</plist>



================================================
FILE: Example/SceneDelegate.swift
================================================
//
//  SceneDelegate.swift
//  Example
//
//  Created by Roman on 28.02.2021.
//

import UIKit

class SceneDelegate: UIResponder, UIWindowSceneDelegate {

    var window: UIWindow?


    func scene(_ scene: UIScene, willConnectTo session: UISceneSession, options connectionOptions: UIScene.ConnectionOptions) {
        // Use this method to optionally configure and attach the UIWindow `window` to the provided UIWindowScene `scene`.
        // If using a storyboard, the `window` property will automatically be initialized and attached to the scene.
        // This delegate does not imply the connecting scene or session are new (see `application:configurationForConnectingSceneSession` instead).
        guard let _ = (scene as? UIWindowScene) else { return }
    }

    func sceneDidDisconnect(_ scene: UIScene) {
        // Called as the scene is being released by the system.
        // This occurs shortly after the scene enters the background, or when its session is discarded.
        // Release any resources associated with this scene that can be re-created the next time the scene connects.
        // The scene may re-connect later, as its session was not necessarily discarded (see `application:didDiscardSceneSessions` instead).
    }

    func sceneDidBecomeActive(_ scene: UIScene) {
        // Called when the scene has moved from an inactive state to an active state.
        // Use this method to restart any tasks that were paused (or not yet started) when the scene was inactive.
    }

    func sceneWillResignActive(_ scene: UIScene) {
        // Called when the scene will move from an active state to an inactive state.
        // This may occur due to temporary interruptions (ex. an incoming phone call).
    }

    func sceneWillEnterForeground(_ scene: UIScene) {
        // Called as the scene transitions from the background to the foreground.
        // Use this method to undo the changes made on entering the background.
    }

    func sceneDidEnterBackground(_ scene: UIScene) {
        // Called as the scene transitions from the foreground to the background.
        // Use this method to save data, release shared resources, and store enough scene-specific state information
        // to restore the scene back to its current state.
    }


}




================================================
FILE: Example/ViewController.swift
================================================
//
//  ViewController.swift
//  Example
//
//  Created by Roman on 28.02.2021.
//

import UIKit
import AVFoundation
import AVKit

class ViewController: UIViewController {

    var observations: [NSObjectProtocol] = []
    private lazy var notificationCenter: NotificationCenter = .default

    override func viewDidLoad() {
        super.viewDidLoad()
    }

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        read()

        observations.append(
            notificationCenter.addObserver(
                forName: UIApplication.willEnterForegroundNotification,
                object: nil,
                queue: nil
            ) { [weak self] _ in
                self?.read()
            }
        )
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)

        observations.forEach(notificationCenter.removeObserver(_:))
    }

    private func read() {
        let fileManager = FileManager.default
        var mediaURLs: [URL] = []
        if let container = fileManager
                .containerURL(
                    forSecurityApplicationGroupIdentifier: "group.com.andrykevych.Example"
                )?.appendingPathComponent("Library/Documents/") {

            let documentsDirectory = fileManager.urls(for: .documentDirectory, in: .userDomainMask).first!
            do {
                let contents = try fileManager.contentsOfDirectory(atPath: container.path)
                for path in contents {
                    guard !path.hasSuffix(".plist") else {
                        print("file at path \(path) is plist, exiting")
                        return
                    }
                    let fileURL = container.appendingPathComponent(path)
                    var isDirectory: ObjCBool = false
                    guard fileManager.fileExists(atPath: fileURL.path, isDirectory: &isDirectory) else {
                        return
                    }
                    guard !isDirectory.boolValue else {
                        return
                    }
                    let destinationURL = documentsDirectory.appendingPathComponent(path)
                    do {
                        try fileManager.copyItem(at: fileURL, to: destinationURL)
                        print("Successfully copied \(fileURL)", "to: ", destinationURL)
                    } catch {
                        print("error copying \(fileURL) to \(destinationURL)", error)
                    }
                    mediaURLs.append(destinationURL)
                }
            } catch {
                print("contents, \(error)")
            }
        }

        mediaURLs.first.map {
            let asset: AVURLAsset = .init(url: $0)
            let item: AVPlayerItem = .init(asset: asset)

            let movie: AVMutableMovie = .init(url: $0)
            for track in movie.tracks {
                print("track", track)
            }

            let player: AVPlayer = .init(playerItem: item)
            let playerViewController: AVPlayerViewController = .init()
            playerViewController.player = player
            present(playerViewController, animated: true, completion: { [player = playerViewController.player] in
                player?.play()
            })
        }
    }
}




================================================
FILE: Example/Assets.xcassets/Contents.json
================================================
{
  "info" : {
    "author" : "xcode",
    "version" : 1
  }
}



================================================
FILE: Example/Assets.xcassets/AccentColor.colorset/Contents.json
================================================
{
  "colors" : [
    {
      "idiom" : "universal"
    }
  ],
  "info" : {
    "author" : "xcode",
    "version" : 1
  }
}



================================================
FILE: Example/Assets.xcassets/AppIcon.appiconset/Contents.json
================================================
{
  "images" : [
    {
      "idiom" : "iphone",
      "scale" : "2x",
      "size" : "20x20"
    },
    {
      "idiom" : "iphone",
      "scale" : "3x",
      "size" : "20x20"
    },
    {
      "idiom" : "iphone",
      "scale" : "2x",
      "size" : "29x29"
    },
    {
      "idiom" : "iphone",
      "scale" : "3x",
      "size" : "29x29"
    },
    {
      "idiom" : "iphone",
      "scale" : "2x",
      "size" : "40x40"
    },
    {
      "idiom" : "iphone",
      "scale" : "3x",
      "size" : "40x40"
    },
    {
      "idiom" : "iphone",
      "scale" : "2x",
      "size" : "60x60"
    },
    {
      "idiom" : "iphone",
      "scale" : "3x",
      "size" : "60x60"
    },
    {
      "idiom" : "ipad",
      "scale" : "1x",
      "size" : "20x20"
    },
    {
      "idiom" : "ipad",
      "scale" : "2x",
      "size" : "20x20"
    },
    {
      "idiom" : "ipad",
      "scale" : "1x",
      "size" : "29x29"
    },
    {
      "idiom" : "ipad",
      "scale" : "2x",
      "size" : "29x29"
    },
    {
      "idiom" : "ipad",
      "scale" : "1x",
      "size" : "40x40"
    },
    {
      "idiom" : "ipad",
      "scale" : "2x",
      "size" : "40x40"
    },
    {
      "idiom" : "ipad",
      "scale" : "1x",
      "size" : "76x76"
    },
    {
      "idiom" : "ipad",
      "scale" : "2x",
      "size" : "76x76"
    },
    {
      "idiom" : "ipad",
      "scale" : "2x",
      "size" : "83.5x83.5"
    },
    {
      "idiom" : "ios-marketing",
      "scale" : "1x",
      "size" : "1024x1024"
    }
  ],
  "info" : {
    "author" : "xcode",
    "version" : 1
  }
}



================================================
FILE: Example/Base.lproj/LaunchScreen.storyboard
================================================
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<document type="com.apple.InterfaceBuilder3.CocoaTouch.Storyboard.XIB" version="3.0" toolsVersion="13122.16" targetRuntime="iOS.CocoaTouch" propertyAccessControl="none" useAutolayout="YES" launchScreen="YES" useTraitCollections="YES" useSafeAreas="YES" colorMatched="YES" initialViewController="01J-lp-oVM">
    <dependencies>
        <plugIn identifier="com.apple.InterfaceBuilder.IBCocoaTouchPlugin" version="13104.12"/>
        <capability name="Safe area layout guides" minToolsVersion="9.0"/>
        <capability name="documents saved in the Xcode 8 format" minToolsVersion="8.0"/>
    </dependencies>
    <scenes>
        <!--View Controller-->
        <scene sceneID="EHf-IW-A2E">
            <objects>
                <viewController id="01J-lp-oVM" sceneMemberID="viewController">
                    <view key="view" contentMode="scaleToFill" id="Ze5-6b-2t3">
                        <rect key="frame" x="0.0" y="0.0" width="375" height="667"/>
                        <autoresizingMask key="autoresizingMask" widthSizable="YES" heightSizable="YES"/>
                        <color key="backgroundColor" xcode11CocoaTouchSystemColor="systemBackgroundColor" cocoaTouchSystemColor="whiteColor"/>
                        <viewLayoutGuide key="safeArea" id="6Tk-OE-BBY"/>
                    </view>
                </viewController>
                <placeholder placeholderIdentifier="IBFirstResponder" id="iYj-Kq-Ea1" userLabel="First Responder" sceneMemberID="firstResponder"/>
            </objects>
            <point key="canvasLocation" x="53" y="375"/>
        </scene>
    </scenes>
</document>



================================================
FILE: Example/Base.lproj/Main.storyboard
================================================
<?xml version="1.0" encoding="UTF-8"?>
<document type="com.apple.InterfaceBuilder3.CocoaTouch.Storyboard.XIB" version="3.0" toolsVersion="13122.16" targetRuntime="iOS.CocoaTouch" propertyAccessControl="none" useAutolayout="YES" useTraitCollections="YES" useSafeAreas="YES" colorMatched="YES" initialViewController="BYZ-38-t0r">
    <dependencies>
        <plugIn identifier="com.apple.InterfaceBuilder.IBCocoaTouchPlugin" version="13104.12"/>
        <capability name="Safe area layout guides" minToolsVersion="9.0"/>
        <capability name="documents saved in the Xcode 8 format" minToolsVersion="8.0"/>
    </dependencies>
    <scenes>
        <!--View Controller-->
        <scene sceneID="tne-QT-ifu">
            <objects>
                <viewController id="BYZ-38-t0r" customClass="ViewController" customModuleProvider="target" sceneMemberID="viewController">
                    <view key="view" contentMode="scaleToFill" id="8bC-Xf-vdC">
                        <rect key="frame" x="0.0" y="0.0" width="375" height="667"/>
                        <autoresizingMask key="autoresizingMask" widthSizable="YES" heightSizable="YES"/>
                        <color key="backgroundColor" xcode11CocoaTouchSystemColor="systemBackgroundColor" cocoaTouchSystemColor="whiteColor"/>
                        <viewLayoutGuide key="safeArea" id="6Tk-OE-BBY"/>
                    </view>
                </viewController>
                <placeholder placeholderIdentifier="IBFirstResponder" id="dkx-z0-nzr" sceneMemberID="firstResponder"/>
            </objects>
        </scene>
    </scenes>
</document>



================================================
FILE: Sources/BroadcastWriter/AVAssetWriter.Status+stringDescription.swift
================================================

import AVFoundation

extension AVAssetWriter.Status: CustomStringConvertible {

    public var description: String {
        switch self {
        case .cancelled:
            return "cancelled"
        case .completed:
            return "completed"
        case .failed:
            return "failed"
        case .unknown:
            return "unknown"
        case .writing:
            return "writing"
        @unknown default:
            return "@unknown default"
        }
    }
}



================================================
FILE: Sources/BroadcastWriter/BroadcastWriter.swift
================================================

import AVFoundation
import ReplayKit

enum Error: Swift.Error {
    case wrongAssetWriterStatus(AVAssetWriter.Status)
    case selfDeallocated
}

public final class BroadcastWriter {

    private var assetWriterSessionStarted: Bool = false
    private let assetWriterQueue: DispatchQueue
    private let assetWriter: AVAssetWriter

    private lazy var videoInput: AVAssetWriterInput = {

        let videoWidth = screenSize.width * screenScale
        let videoHeight = screenSize.height * screenScale

        let compressionProperties: [String: Any] = [
            AVVideoExpectedSourceFrameRateKey: 60.nsNumber,
            AVVideoProfileLevelKey: "HEVC_Main_AutoLevel"
        ]
        let videoSettings: [String: Any] = [
            AVVideoCodecKey: AVVideoCodecType.hevc,
            AVVideoWidthKey: videoWidth.nsNumber,
            AVVideoHeightKey: videoHeight.nsNumber,
            AVVideoCompressionPropertiesKey: compressionProperties
        ]
        let input: AVAssetWriterInput = .init(
            mediaType: .video,
            outputSettings: videoSettings
        )
        input.expectsMediaDataInRealTime = true
        return input
    }()

    private var audioSampleRate: Double {
        AVAudioSession.sharedInstance().sampleRate
    }
    private lazy var audioInput: AVAssetWriterInput = {

        var audioSettings: [String: Any] = [
            AVFormatIDKey: kAudioFormatMPEG4AAC,
            AVNumberOfChannelsKey: 1,
            AVSampleRateKey: audioSampleRate
        ]
        let input: AVAssetWriterInput = .init(
            mediaType: .audio,
            outputSettings: audioSettings
        )
        input.expectsMediaDataInRealTime = true
        return input
    }()

    private lazy var microphoneInput: AVAssetWriterInput = {
        let sampleRate = AVAudioSession.sharedInstance().sampleRate

        var audioSettings: [String: Any] = [
            AVFormatIDKey: kAudioFormatMPEG4AAC,
            AVNumberOfChannelsKey: 1,
            AVSampleRateKey: audioSampleRate
        ]
        let input: AVAssetWriterInput = .init(
            mediaType: .audio,
            outputSettings: audioSettings
        )
        input.expectsMediaDataInRealTime = true
        return input
    }()

    private lazy var inputs: [AVAssetWriterInput] = [
        videoInput,
        audioInput,
        microphoneInput
    ]

    private let screenSize: CGSize
    private let screenScale: CGFloat

    public init(
        outputURL url: URL,
        assetWriterQueue queue: DispatchQueue = .init(label: "BroadcastSampleHandler.assetWriterQueue"),
        screenSize: CGSize,
        screenScale: CGFloat
    ) throws {
        assetWriterQueue = queue
        assetWriter = try .init(url: url, fileType: .mp4)

        self.screenSize = screenSize
        self.screenScale = screenScale
    }

    public func start() throws {
        try assetWriterQueue.sync {
            let status = assetWriter.status
            guard status == .unknown else {
                throw Error.wrongAssetWriterStatus(status)
            }
            try assetWriter.error.map {
                throw $0
            }
            inputs
                .lazy
                .filter(assetWriter.canAdd(_:))
                .forEach(assetWriter.add(_:))
            try assetWriter.error.map {
                throw $0
            }
            assetWriter.startWriting()
            try assetWriter.error.map {
                throw $0
            }
        }
    }

    public func processSampleBuffer(
        _ sampleBuffer: CMSampleBuffer,
        with sampleBufferType: RPSampleBufferType
    ) throws -> Bool {

        guard sampleBuffer.isValid,
              CMSampleBufferDataIsReady(sampleBuffer) else {
            debugPrint(
                "sampleBuffer.isValid", sampleBuffer.isValid,
                "CMSampleBufferDataIsReady(sampleBuffer)", CMSampleBufferDataIsReady(sampleBuffer)
            )
            return false
        }

        let isWriting = assetWriterQueue.sync {
            assetWriter.status == .writing
        }

        guard isWriting else {
            debugPrint(
                "assetWriter.status",
                assetWriter.status.description,
                "assetWriter.error:",
                assetWriter.error  ?? "no error"
            )
            return false
        }

        assetWriterQueue.sync {
            startSessionIfNeeded(sampleBuffer: sampleBuffer)
        }

        let capture: (CMSampleBuffer) -> Bool
        switch sampleBufferType {
        case .video:
            capture = captureVideoOutput
        case .audioApp:
            capture = captureAudioOutput
        case .audioMic:
            capture = captureMicrophoneOutput
        @unknown default:
            debugPrint(#file, "Unknown type of sample buffer, \(sampleBufferType)")
            capture = { _ in false }
        }

        return assetWriterQueue.sync {
            capture(sampleBuffer)
        }
    }

    public func pause() {
        // TODO: Pause
    }

    public func resume() {
        // TODO: Resume
    }

    public func finish() throws -> URL {
        return try assetWriterQueue.sync {
            let group: DispatchGroup = .init()

            inputs
                .lazy
                .filter { $0.isReadyForMoreMediaData }
                .forEach { $0.markAsFinished() }

            let status = assetWriter.status
            guard status == .writing else {
                throw Error.wrongAssetWriterStatus(status)
            }
            group.enter()

            var error: Swift.Error?
            assetWriter.finishWriting { [weak self] in

                defer {
                    group.leave()
                }

                guard let self = self else {
                    error = Error.selfDeallocated
                    return
                }

                if let e = self.assetWriter.error {
                    error = e
                    return
                }

                let status = self.assetWriter.status
                guard status == .completed else {
                    error = Error.wrongAssetWriterStatus(status)
                    return
                }
            }
            group.wait()
            try error.map { throw $0 }
            return assetWriter.outputURL
        }
    }
}

private extension BroadcastWriter {

    func startSessionIfNeeded(sampleBuffer: CMSampleBuffer) {
        guard !assetWriterSessionStarted else {
            return
        }

        let sourceTime = CMSampleBufferGetPresentationTimeStamp(sampleBuffer)
        assetWriter.startSession(atSourceTime: sourceTime)
        assetWriterSessionStarted = true
    }

    func captureVideoOutput(_ sampleBuffer: CMSampleBuffer) -> Bool {
        guard videoInput.isReadyForMoreMediaData else {
            debugPrint("audioInput is not ready")
            return false
        }
        return videoInput.append(sampleBuffer)
    }

    func captureAudioOutput(_ sampleBuffer: CMSampleBuffer) -> Bool {
        guard audioInput.isReadyForMoreMediaData else {
            debugPrint("audioInput is not ready")
            return false
        }
        return audioInput.append(sampleBuffer)
    }

    func captureMicrophoneOutput(_ sampleBuffer: CMSampleBuffer) -> Bool {

        guard microphoneInput.isReadyForMoreMediaData else {
            debugPrint("microphoneInput is not ready")
            return false
        }
        return microphoneInput.append(sampleBuffer)
    }
}



================================================
FILE: Sources/BroadcastWriter/CGFloat+NSNumber.swift
================================================

import Foundation
import CoreGraphics

extension CGFloat {

    var nsNumber: NSNumber {
        return .init(value: native)
    }
}



================================================
FILE: Sources/BroadcastWriter/Int+NSNumber.swift
================================================

import Foundation

extension Int {
    
    var nsNumber: NSNumber {
        return .init(value: self)
    }
}



================================================
FILE: Tests/LinuxMain.swift
================================================
import XCTest

import BroadcastWriterTests

var tests = [XCTestCaseEntry]()
tests += BroadcastWriterTests.allTests()
XCTMain(tests)



================================================
FILE: Tests/BroadcastWriterTests/BroadcastWriterTests.swift
================================================
import XCTest
@testable import BroadcastWriter

final class BroadcastWriterTests: XCTestCase {
    func testExample() {
        // This is an example of a functional test case.
        // Use XCTAssert and related functions to verify your tests produce the correct
        // results.
//        XCTAssertEqual(BroadcastSampleHandler().text, "Hello, World!")
    }

    static var allTests = [
        ("testExample", testExample),
    ]
}



================================================
FILE: Tests/BroadcastWriterTests/XCTestManifests.swift
================================================
import XCTest

#if !canImport(ObjectiveC)
public func allTests() -> [XCTestCaseEntry] {
    return [
        testCase(BroadcastWriterTests.allTests),
    ]
}
#endif


================================================
FILE: README.md
================================================
# Broadcast-Upload-Extension
A simple broadcast upload extension to help you test your ReplayKit code before app review.

## Why you are here
You are trying to use ReplayKit in an iOS app and App Review is rejecting you because they expect you to also implement a Broadcast Upload Extension but refuse to tell you that it's a requirement.  This contradicts the spirit of app extensions, you're irritated and they keep suggesting that you look to the Apple Developer Forums where you find answers like this https://developer.apple.com/forums/thread/60566.

Spoiler alert, the post is 5+ years old with zero replies.

## How to solve your issue
This is a simple implementation of a Broadcast Upload Extension that encodes your stream and saves it as an mp4 into the Photos Library.

I recommend that you title your extensions with simple names like 'Record' or 'Local' or some other simple, short title to help convey what they do without the horrible word-wrapping that occurs in Apple's UI.

1. First, create a Broadcast Upload Extension target in your app. You will also need a Broadcast Upload Setup UI target so you can do that at the same time or later.

* The title of the Broadcast Upload Extension will show up in the list of extensions when you long press the screen record button in the iOS control pannel.
* The title of the Setup UI extension will show up in the RPBroadcastActivityViewController, but you end up at the same block of code either way.

2. Add PhotoAddUsageDescription to the Info.plist in the Upload Extension
3. Use swift package manager to include git@github.com:romiroma/BroadcastWriter.git for your Upload Extension.
4. Copy the code from the .swift files here into your corresponding files in the new targets you created.
5. Replace the indicated strings in your code.
6. Test to make sure it works!

## Thank You!
This code is a compilation of solutions I was able to find across Stack Overflow, GitHub and other random places.
https://github.com/romiroma/BroadcastWriter/commits?author=romiroma provided a great starting place, thank you for that!
And of course thanks goes out to everyone on the internet that tried to explain how this worked, except Apple who did a horrible job at it.



================================================
FILE: BroadcastSetupViewController.swift
================================================
import ReplayKit

class BroadcastSetupViewController: UIViewController {

  // MARK: Replace These Values
  let broadcastName = "My Recording"
  let myErrorDomain = "My Error Domain"

  private let fileManager: FileManager = .default

  override func viewDidAppear(_ animated: Bool) {
    userDidFinishSetup()
  }

  // Call this method when the user has finished interacting with the view controller and a broadcast stream can start
  func userDidFinishSetup() {
    // URL of the resource where broadcast can be viewed that will be returned to the application
    let broadcastURL = fileManager.temporaryDirectory
      .appendingPathComponent(UUID().uuidString)
      .appendingPathExtension(for: .mpeg4Movie)

    // Dictionary with setup information that will be provided to broadcast extension when broadcast is started
    let setupInfo: [String : NSCoding & NSObjectProtocol] = ["broadcastName": broadcastName as NSCoding & NSObjectProtocol]

    // Tell ReplayKit that the extension is finished setting up and can begin broadcasting
    self.extensionContext?.completeRequest(withBroadcast: broadcastURL, setupInfo: setupInfo)
  }

  func userDidCancelSetup() {
    let error = NSError(domain: myErrorDomain, code: -1, userInfo: nil)
    // Tell ReplayKit that the extension was cancelled by the user
    self.extensionContext?.cancelRequest(withError: error)
  }
}



================================================
FILE: SampleHandler.swift
================================================
import BroadcastWriter
import ReplayKit
import Photos

class SampleHandler: RPBroadcastSampleHandler {

  private var writer: BroadcastWriter?
  private let fileManager: FileManager = .default
  private let nodeURL: URL

  private let videoOutputFullFileName = "\(Date().timeIntervalSince1970)"

  override init() {
    print("🧩 Broadcast Sample Handler Created")
    nodeURL = fileManager.temporaryDirectory
      .appendingPathComponent(UUID().uuidString)
      .appendingPathExtension(for: .mpeg4Movie)

    fileManager.removeFileIfExists(url: nodeURL)

    super.init()
  }

  override func broadcastStarted(withSetupInfo setupInfo: [String : NSObject]?) {
    let screen: UIScreen = .main
    do {
      writer = try .init(
        outputURL: nodeURL,
        screenSize: screen.bounds.size,
        screenScale: screen.scale
      )
    } catch {
      assertionFailure(error.localizedDescription)
      finishBroadcastWithError(error)
      return
    }
    do {
      try writer?.start()
    } catch {
      finishBroadcastWithError(error)
    }
  }

  override func processSampleBuffer(_ sampleBuffer: CMSampleBuffer, with sampleBufferType: RPSampleBufferType) {
    guard let writer = writer else {
      debugPrint("processSampleBuffer: Writer is nil")
      return
    }

    do {
      let captured = try writer.processSampleBuffer(sampleBuffer, with: sampleBufferType)
      debugPrint("processSampleBuffer captured", captured)
    } catch {
      debugPrint("processSampleBuffer error:", error.localizedDescription)
    }
  }

  override func broadcastPaused() {
    debugPrint("=== paused")
    writer?.pause()
  }

  override func broadcastResumed() {
    debugPrint("=== resumed")
    writer?.resume()
  }

  override func broadcastFinished() {
    guard let writer = writer else {
      return
    }

    let dispatchGroup = DispatchGroup()
    dispatchGroup.enter()
    let outputURL: URL
    do {
      outputURL = try writer.finish()
    } catch {
      debugPrint("writer failure", error)
      dispatchGroup.leave()
      return
    }

    PHPhotoLibrary.shared().performChanges({
      PHAssetChangeRequest.creationRequestForAssetFromVideo(atFileURL: outputURL)
    }) { completed, error in
      if completed {
        print("🎥 Video \(self.videoOutputFullFileName) has been moved to camera roll")
      }

      if error != nil {
        print ("🧨 Cannot move the video \(self.videoOutputFullFileName) to camera roll, error: \(error!.localizedDescription)")
      }
      dispatchGroup.leave()
    }
    dispatchGroup.wait() // <= blocks the thread here
  }
}

extension FileManager {

  func removeFileIfExists(url: URL) {
    guard fileExists(atPath: url.path) else { return }
    do {
      try removeItem(at: url)
    } catch {
      print("error removing item \(url)", error)
    }
  }
}


