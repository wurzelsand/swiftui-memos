## @Published enum verschickt Nachricht

Wir haben eine `ObservableObject`-Klasse `UserList` die eine `@Published` Eigenschaft `ordering` besitzt. `ordering` ist ein Enumerationstyp. Wenn er seinen Wert ändert, soll über `print` darüber informiert werden.

```swift
import SwiftUI
import Combine

class UserList: ObservableObject {
    enum Ordering {
        case byName
        case byAge
    }
    
    @Published var ordering: Ordering = .byName
}

struct ContentView: View {
    @StateObject var userList = UserList()
    
    var body: some View {
        Button("Toggle") {
            switch userList.ordering {
            case .byAge:
                userList.ordering = .byName
            case .byName:
                userList.ordering = .byAge
            }
        }
    }
}
```

Der Button soll so gelassen werden: Er ändert nur den Wert von `ordering`. Die `print`-Nachricht soll aus der `UserList`-Klasse verschickt werden.

```swift
import SwiftUI
import Combine

class UserList: ObservableObject {
    enum Ordering {
        case byName
        case byAge
    }
    
    @Published var ordering: Ordering = .byName
    private var cancellable: Cancellable?
    
    init() {
        cancellable = $ordering.map { ordering -> String in
            switch ordering {
            case .byAge:
                return "ordering by age"
            case .byName:
                return "ordering by name"
            }
        }.sink { value in
            print(value)
        }
    }
}

struct ContentView: View {
    @StateObject var userList = UserList()
    
    var body: some View {
        Button("Toggle") {
            switch userList.ordering {
            case .byAge:
                userList.ordering = .byName
            case .byName:
                userList.ordering = .byAge
            }
        }
    }
}
```
