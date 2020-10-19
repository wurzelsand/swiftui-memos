## Combine: Ausgabe wie bei reduce()

### Aufgabe

Wir haben eine Sequenz:

```swift
let range = (0...5)
```

Wir wollen jedes Element der Sequenz addieren und dabei die Zwischenergebnisse anzeigen. Üblich wäre z. B.:

```swift
range.reduce(0) { (result, element) in
    let nextResult = result + element
    print("\(nextResult)", terminator: " ")
    return nextResult
}
```

Ausgabe:

<pre><code>0 1 3 6 10 15 </pre></code>

Wenn wir `Combine` importieren, werden Sequenzen automatisch um mehrere Methoden erweitert. Eine davon entspricht der `reduce`-Methode.

## Ausführung:

```swift
import Combine

let range = (0...5)
let cancellable = range.publisher
    .scan(0) {
        return $0 + $1
    }
    .sink { print("\($0)", terminator: " ") }
 ```
 
 Ausgabe:
 
<pre><code>0 1 3 6 10 15 </pre></code>
