## GRDB

### Zunächst die einfachste App mit GRDB

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

Der `DatabaseWriter` kümmert sich um das Schreiben der Datenbank auf Dateiebene. Wir werden in *Persistence.swift* zwei verschiedene `DatabaseWriter` erstellen: Einen der seine Daten tatsächlich dauerhaft in einer Datei speichert und einen, der die Daten nur temporär im Arbeitsspeicher ablegt und der sich damit für Previews eignet.

Der `DatabaseMigrator` erstellt unsere Datenbank-Tabelle und könnte später unsere Datenbank migrieren, wenn wir uns entschließen die Datenbank zu ändern.

Die Methoden `saveItem` und `deleteItems` delegieren ihre Schreibvorgänge an die `Item`-Objekte.

Die Methode `itemsPublisher` ist das eigentliche Herzstück unserer Datenbank. Es erstellt einen Publisher, der mit `ValueObservation.tracking` Änderungen an `Item`-Objekten in der Datenbank überwacht und diese an seine Subscriber übermittelt. Unser `ItemListModel` benutzt einen solchen Subscriber um seine Liste aus `Item`-Objekten auf dem aktuellen Stand zu halten.

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
