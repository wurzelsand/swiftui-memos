## Struct als Parameter übergeben und verändernde Methode aufrufen

Wir haben eine Struktur `Player` die über die Methode `nextLevel` ihr `score` aufwerten kann:

```swift
struct Player {
    private(set) var score: Int
    mutating func nextLevel() {
        score += 1
    }
}
```

Die Funktion `levelCompleted` soll `Player` aufwerten:

```swift
func levelCompleted(player: Player) {
    player.nextLevel() // Error!
}

var player = Player(score: 0)
levelCompleted(player: player)
```

Fehler:

<pre><code>Cannot use mutating member on immutable value: 'player' is a 'let' constant</pre></code>

### Ausführung

```swift
struct Player {
    private(set) var score: Int
    mutating func nextLevel() {
        score += 1
    }
}

func levelCompleted(player: inout Player) {
    player.nextLevel()
}

var player = Player(score: 0)
levelCompleted(player: &player)
```
