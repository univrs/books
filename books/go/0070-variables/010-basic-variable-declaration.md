Title: Basic Variable Declaration
Id: 2192
Score: 6
Body:
Go is a statically typed language, meaning you generally have to declare the type of the variables you are using.

    // Basic variable declaration. Declares a variable of type specified on the right.
    // The variable is initialized to the zero value of the respective type.
    var x int
    var s string
    var p Person // Assuming type Person struct {}

    // Assignment of a value to a variable
    x = 3

    // Short declaration using := infers the type
    y := 4

    u := int64(100)    // declare variable of type int64 and init with 100 
    var u2 int64 = 100 // declare variable of type int64 and init with 100

|======|