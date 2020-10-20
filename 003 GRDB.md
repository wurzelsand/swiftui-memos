## GRDB

### Zunächst die einfachste App mit GRDB

Wir benutzen GRDB als SQLite-Wrapper und erstellen eine möglichst simple App: Die Datenbank soll aus einer Liste von Items bestehen. Jedes `Item` besteht aus einem Namen (`name`) und optional aus seiner Anzahl (`quantity`). Der Button *New* erstellt ein neues Item mit einem zufälligen Namen und einer zufälligen Anzahl. Durch ein Wischen nach links können Items aus der Liste gelöscht werden.

<a><img src="media/simplest-grdb-app.gif" height=400><a>
