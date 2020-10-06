## Liste von Namen editieren

**Aufgabe:** Eine Firma (`Company`) besteht aus drei Angestellten (`Employee`). Die Namen der Angestellten sollen in einem `ForEach`-*View* dargestellt werden. Wird auf einen Angestellten geklickt, kann man dessen Namen in einem neuen Fenster editieren.

<img src="media/edit-list-of-employees.gif" width=300>

### Version 1

```swift
import SwiftUI

class Employee: Identifiable {
    var name: String
    
    init(_ name: String) {
        self.name = name
    }
}

class Company: ObservableObject {
    var employees = [Employee]()
    
    init(members: String...) {
        for member in members {
            self.employees.append(Employee(member))
        }
    }
}

struct ContentView: View {
    @State private var editEmployee: Employee?
    @StateObject var company = Company(members: "Sam", "Tom", "Jim")
    
    var body: some View {
        ForEach(company.employees) { employee in
            Text(employee.name).onTapGesture {
                editEmployee = employee
            }
        }.sheet(item: $editEmployee) { employee in
            EditView(employee: employee)
                .environmentObject(company)
        }
    }
}

struct EditView: View {
    @Environment(\.presentationMode) var presentation
    @EnvironmentObject var company: Company
    
    var employee: Employee
    @State private var nameEdit: String
    
    init(employee: Employee) {
        self.employee = employee
        self._nameEdit = State(initialValue: employee.name)
    }
    
    var body: some View {
        TextField("Employee", text: $nameEdit)
        Button("Save") {
            company.objectWillChange.send()
            employee.name = nameEdit
            presentation.wrappedValue.dismiss()
        }
    }
}
```

`Employee` muss ein Referenztyp sein, da wir ansonsten in `EditView` die Eigenschaft `employee` nicht verändern könnten (*'self' is immutable*). Das Problem ist, dass die Änderung der `name`-Eigenschaft von `employee` von `ContentView` nicht bemerkt wird. Der neue Name würde dann nicht angezeigt. Daher rufe ich explizit `company.objectWillChange.send()` auf.

### Version 2

Mich störte `company.objectWillChange.send()`. Daher habe ich die Lösung verändert. 
* Die Eigenschaft `employees` von `EditView` ist wegen `@Binding` nun veränderbar.
* Um den Namen eines Angestellten zu verändern, benötigen wir den Index des Angestellen innerhalb des Arrays.
* Um den Index als `item` für `sheet` benutzen zu können muss der Typ des Index das `Identifiable`-Protokoll erfüllen. Daher die Klasse `SelectedIndex`. Möglich wäre es auch, eine `Identifiable`-*Extension* für `Int` zu verwenden.

```swift
import SwiftUI

class SelectedIndex: Identifiable {
    let idx: Int
    
    init(_ idx: Int) {
        self.idx = idx
    }
}

class Company: ObservableObject {
    @Published var employees = ["Sam", "Tom", "Jim"]
}

struct ContentView: View {
    @State private var selection: SelectedIndex?
    @StateObject var company = Company()
    
    var body: some View {
        ForEach(company.employees.indices) { idx in
            Text(company.employees[idx]).onTapGesture {
                selection = SelectedIndex(idx)
            }
        }.sheet(item: $selection) { selected in
            EditView(employees: $company.employees, idx: selected.idx)
        }
    }
}

struct EditView: View {
    @Environment(\.presentationMode) var presentation
    
    @Binding var employees: [String]
    let idx: Int
    @State private var nameEdit: String
    
    init(employees: Binding<[String]>, idx: Int) {
        self._employees = employees
        self.idx = idx
        self._nameEdit = State(initialValue: employees.wrappedValue[idx])
    }
    
    var body: some View {
        TextField("Employee", text: $nameEdit)
        Button("Save") {
            employees[idx] = nameEdit
            presentation.wrappedValue.dismiss()
        }
    }
}
```

### Version 3

Dass man in Version 2 über die Indizes gehen muss, finde ich auch nicht so ideal. Deshalb reaktiviere ich noch einmal `objectWillChange.send()`, aber diesmal innerhalb der `Employee`-Klasse, so dass es nur an einer Stelle stehen muss und nicht an jeder Stelle, an der `Employee` verändert wird.

```swift
import SwiftUI

class Employee: Identifiable {
    var name: String {
        willSet {
            company.objectWillChange.send()
        }
    }
    var company: Company
    
    init(name: String, company: Company) {
        self.name = name
        self.company = company
    }
}

class Company: ObservableObject {
    var employees = [Employee]()
    
    init(employees: String...) {
        for name in employees {
            self.employees.append(Employee(name: name, company: self))
        }
    }
}

struct ContentView: View {
    @State private var selected: Employee?
    @StateObject var company = Company(employees: "Sam", "Tom", "Jim")
    
    var body: some View {
        ForEach(company.employees) { employee in
            Text(employee.name).onTapGesture {
                selected = employee
            }
        }.sheet(item: $selected) { selected in
            EditView(employee: selected)
        }
    }
}

struct EditView: View {
    @Environment(\.presentationMode) var presentation
    
    var employee: Employee?
    @State private var nameEdit: String
    
    init(employee: Employee) {
        self.employee = employee
        self._nameEdit = State(initialValue: employee.name)
    }
    
    var body: some View {
        TextField("Employee", text: $nameEdit)
        Button("Save") {
            employee?.name = nameEdit
            presentation.wrappedValue.dismiss()
        }
    }
}
```
