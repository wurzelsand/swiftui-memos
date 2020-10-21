## GRDB

### Zunächst eine möglichst minimale App mit GRDB

Wir benutzen GRDB als SQLite-Wrapper und erstellen eine möglichst simple App: Die Datenbank soll aus einer Liste von Items bestehen. Jedes `Item` besteht aus einem Namen (`name`) und optional aus seiner Anzahl (`quantity`). Der Button *New* erstellt ein neues Item mit einem zufälligen Namen und einer zufälligen Anzahl. Durch ein Wischen nach links können Items aus der Liste gelöscht werden.

<a><img src="media/simplest-grdb-app.gif" height=400><a>

### Ausführung

<a><img src="media/grdbsmall-files.png" width=300><a>

*GRDBSmallApp.swift:*

Das Eintritts-View nennen wir `ItemListView`. Sein korrespondierendes ViewModel nennen wir `ItemListModel`, das pro Szene mit dem Singleton einer Datenbank-Struktur `AppDatabase` initialisiert wird.

```swift
import SwiftUI

@main
struct GRDBSmallApp: App {
    let appDatabase = AppDatabase.shared
    
    var body: some Scene {
        WindowGroup {
            ItemListView(itemListModel: ItemListModel(database: appDatabase))
        }
    }
}
```

*AppDatabase.swift:*

```swift
import GRDB
import Combine

struct AppDatabase {
    private let databaseWriter: DatabaseWriter
    
    init(_ databaseWriter: DatabaseWriter) throws {
        self.databaseWriter = databaseWriter
        try migrator.migrate(databaseWriter)
    }
    
    private var migrator: DatabaseMigrator {
        var migrator = DatabaseMigrator()
        #if DEBUG
        migrator.eraseDatabaseOnSchemaChange = true
        #endif
        migrator.registerMigration("createItem") { database in
            try database.create(table: "item") { tableDefinition in
                tableDefinition.autoIncrementedPrimaryKey("id")
                tableDefinition.column("name", .text).notNull()
                tableDefinition.column("quantity", .integer)
            }
        }
        return migrator
    }
}

extension AppDatabase {
    func saveItem(_ item: inout Item) throws {
        try databaseWriter.write { database in
            try item.save(database)
        }
    }
    
    func deleteItems(ids: [Int64]) throws {
        try databaseWriter.write { database in
            _ = try Item.deleteAll(database, keys: ids)
        }
    }
    
    func itemsPublisher() -> AnyPublisher<[Item], Error> {
        ValueObservation.tracking(Item.all().fetchAll)
            .publisher(in: databaseWriter)
            .eraseToAnyPublisher()
    }
}
```

Der `DatabaseWriter` kümmert sich um das Schreiben der Datenbank auf Dateiebene. Wir werden später in *Persistence.swift* zwei verschiedene `DatabaseWriter` erstellen: Einen der seine Daten tatsächlich dauerhaft in einer Datei speichert und einen, der die Daten nur temporär im Arbeitsspeicher ablegt und der sich damit für Previews eignet.

Der `DatabaseMigrator` erstellt unsere Datenbank-Tabelle und könnte später unsere Datenbank migrieren, wenn wir uns entschließen die Datenbank zu ändern.

Die Methoden `saveItem` und `deleteItems` delegieren ihre Schreibvorgänge an die `Item`-Objekte.

Die Methode `itemsPublisher` ist das eigentliche Herzstück unserer Datenbank. Es erstellt einen Publisher, der mit `ValueObservation.tracking` Änderungen an `Item`-Objekten in der Datenbank überwacht und diese an seine Subscriber übermittelt. Unser `ItemListModel` benutzt einen solchen Subscriber um seine Liste aus `Item`-Objekten auf dem aktuellen Stand zu halten. Eine ausführlichere Deklaration von `itemsPublisher` wäre:

```swift
func itemsPublisher() -> AnyPublisher<[Item], Error> {
    let valueObservation = ValueObservation.tracking
    { (database: Database) throws -> [Item] in
        try Item.all().fetchAll(database)
    }
    return valueObservation.publisher(in: databaseWriter)
        .eraseToAnyPublisher()
}
```

`valueObservation` ist dabei ein Objekt, das dem Closure Zugriff auf eine Datenbank gibt. Im Closure erstellt die `fetchAll`-Funktion ein Array aus sämtlichen *Items*, die in der Datenbank gefunden werden. `valueObservation` kann nun dieses Array überwachen. Den Zugriff auf die Datenbank bekommt `valueObservation` aber erst im nächsten Schritt: Hier wird das Closure mit `databaseWriter` verknüpft und ergibt einen *Publisher*. [swift-memos 004](https://github.com/wurzelsand/swift-memos/blob/main/004%20ValueObservation%20aus%20GRDB%20verstehen.md)

*Persistence.swift:*

```swift
import GRDB

extension AppDatabase {
    static let shared = makeShared()
    
    private static func makeShared() -> AppDatabase {
        do {
            let url = try FileManager.default.url(
                for: .applicationSupportDirectory,
                in: .userDomainMask,
                appropriateFor: nil,
                create: true)
                .appendingPathComponent("db.sqlite")
            let databasePool = try DatabasePool(path: url.path)
            let appDatabase = try AppDatabase(databasePool)
            return appDatabase
        } catch {
            fatalError("Unresolved error \(error)")
        }
    }
    
    static func empty() throws -> AppDatabase {
        let dbQueue = DatabaseQueue()
        return try AppDatabase(dbQueue)
    }
}
```

`shared` ist das Singleton von `AppDatabase`, welches die Daten seiner Datenbank in einer Datei dauerhaft speichert. `empty` ist eigentlich nur ein leerer `AppDatabase`-Platzhalter für Previews.

*Item.swift:*

```swift
import GRDB

struct Item: Identifiable {
    var id: Int64?
    var name: String
    var quantity: Int?
}

extension Item {
    static func new() -> Item {
        Item(id: nil, name: "", quantity: nil)
    }
}

extension Item: Codable, FetchableRecord, MutablePersistableRecord {
    mutating func didInsert(with rowID: Int64, for column: String?) {
        id = rowID
    }
}

extension Item: CustomStringConvertible {
    var description: String {
        "id: \(id.map { String($0) } ?? "nil"), " +
        "name: \(name), " +
        "quantity: \(quantity.map { String($0) } ?? "nil")"
    }
}
```

Die drei Eigenschaften `id`, `name` und `quantity` entsprechen den gleichlautenden Feldern der SQLite-Datenbank. `id` ist ein *Optional*, da sich das entsprechende Feld der SQLite-Datenbank selbst organisiert und zunächst mit `nil` initialisiert wird.

Die entscheidende *Extension* ist das Erfüllen der Protokolle `Codable`, `FetchableRecord` und `MutablePersistableRecord`.
* `Codable`: Item kann sich in eine externe Represäntation umwandeln und sich auch wieder selbst herstellen.
* `FetchableRecord`: 
