## Liste von Namen editieren

**Aufgabe:** Eine Firma (`Company`) besteht aus drei Angestellten (`Employee`). Die Namen der Angestellten sollen in einem `ForEach`-*View* dargestellt werden. Wird auf einen Angestellten geklickt, kann man dessen Namen in einem neuen Fenster editieren.

<img src="media/edit-list-of-employees.gif" width=300>

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
