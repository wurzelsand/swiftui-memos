## Title

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
