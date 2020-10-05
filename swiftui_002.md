## Ausrichtung des iPhones anzeigen

**Aufgabe:** Beim Start der App soll die Ausrichtung des Geräts angezeigt werden. Beim Drehen soll die Anzeige aktualisiert werden.

<a href="url"><img src="media/orientations.gif" width=300></a>

Ein Problem ist, dass es zwei verschiedene Typen für die Ausrichtung gibt: `UIDeviceOrientation` und `UIInterfaceOrientation`. Dabei sind die Definitionen von *Landscape Right* und *Landscape Left* seltsamerweise vertauscht. Ein weiteres Problem ist, dass sich der Anfangswert von `UIInterfaceOrientation` gut ermitteln lässt, ich es aber ohne die `AppDelegate`-Klasse nicht schaffe, deren Änderungen zu verfolgen. `UIDeviceOrientation` hingegen kann gut bei Änderungen gemessen werden, ich konnte aber keinen Startwert ermitteln. Hier mische ich die Methoden und gebe als Ergebnis `UIInterfaceOrientation` aus.

*@main:*

```swift
import SwiftUI

class InterfaceInfo: ObservableObject {
    @Published var orientation: UIInterfaceOrientation
    private var deviceOrientationObserver: NSObjectProtocol?
    private var sceneActivationObserver: NSObjectProtocol?
    
    init() {
        orientation = .unknown
        
        // unowned: NotificationCenter should not hold a strong reference to InterfaceInfo.
        // NotificationCenter should release InterfaceInfo when Application ends.
        deviceOrientationObserver = NotificationCenter.default.addObserver(
            forName: UIDevice.orientationDidChangeNotification, object: nil, queue: nil)
        { [unowned self] notification in
            if let device = notification.object as? UIDevice {
                switch device.orientation {
                case .portrait:
                    self.orientation = .portrait
                case .portraitUpsideDown:
                    self.orientation = .portraitUpsideDown
                case .landscapeLeft:
                    self.orientation = .landscapeRight // UIDeviceOrientation inverted!
                case .landscapeRight:
                    self.orientation = .landscapeLeft // UIDeviceOrientation inverted!
                default:
                    break
                }
            }
        }
        
        sceneActivationObserver = NotificationCenter.default.addObserver(
            forName: UIScene.didActivateNotification, object: nil, queue: nil)
        { [unowned self] notification in
            if let scene = notification.object as? UIWindowScene {
                self.orientation = scene.interfaceOrientation
            }
        }
    }
    
    deinit {
        if let observer = deviceOrientationObserver {
            NotificationCenter.default.removeObserver(observer)
        }
        if let observer = sceneActivationObserver {
            NotificationCenter.default.removeObserver(observer)
        }
    }
}

@main
struct orientationApp: App {
    var deviceInfo = InterfaceInfo()
    
    var body: some Scene {
        WindowGroup {
            ContentView().environmentObject(deviceInfo)
        }
    }
}
```

*ContentView.swift:*

```swift
import SwiftUI

struct ContentView: View {
    @EnvironmentObject var interfaceInfo: InterfaceInfo
    
    var body: some View {
        Form {
            Section(header: Text("Orientation")) {
                switch interfaceInfo.orientation {
                case .portrait:
                    Text("Portrait")
                case .portraitUpsideDown:
                    Text("Portrait Upside Down")
                case .landscapeLeft:
                    Text("Landscape Left")
                case .landscapeRight:
                    Text("Landscape Right")
                default:
                    Text("Unknown")
                }
            }
        }
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
    }
}
```
