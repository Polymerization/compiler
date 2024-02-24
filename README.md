# A tour of my language

## About

Statically typed language, influenced by zig, rust, go. Semantically most similar to zig (i.e. c). I pulled expression syntax/style from rust. Looked at go for minimalism (e.g. it has no enums). The compiler transforms source files into x86/linux assembly. Use your favourite assembler/linker to produce an exe (depends on c std lib).

usage: 

```
compiler <source file> > asm.s
cc -o program asm.s
```

Only supports gnu as syntax, x86-64 cpu arch, linux os.

## Hello world

```zig
fn main() void {
    print "Hello world!".ptr;
}
```

## Imports, Variables, Functions

```zig
// FILE: math.code
fn sum(a: int, b: int) int {
    return a + b;
}

// globals must have type specified, and can be optionally initialised
var PI: int = 3;

// top level symbols starting with '_' cannot be accessed outside the module
fn _areaOfCircle(radius: int) int {
    PI * radius * radius
}

// FILE: main.code
// paths are relative to the current file
import math "math.code";

fn main() void {
    // type is inferred
    var result = math.sum(1300, 37);

    // value is uninitialised
    var something: int;

    something = multiply(result, something);

    print result;
    print something;
    print math.PI;
}

// all function inputs/outputs are copied by value
fn multiply(a: int, b: int) int {
    // You can optionally specify type when delcaring a variable
    var result: int = a * b;
    return result;
}
```

## Control Flow: If, While

```zig
fn main () void {
    // Similar to rust, most things are an expression 

    // If
    var rand_num: int;
    print rand_num;
    if (rand_num == 4) {
        print "the number is not random".ptr;
    };

    // 'If' are expressions, so they can return values. The final expression in a 
    // block is the return value of the expression. The type of the 'then' and 'else'
    // branch must be the same
    var num = if (rand_num == 0) {
        print "then branch\n".ptr;
        66
    } else {
        print "else branch".ptr;
        883
    };
    print num;

    // If you return in a If branch, it uses the other branch to determine the type
    var another_num = if (rand_num == 44) {
        return;
    } else {
        3
    };
    print another_num;
    
    // While
    var i = 0;
    while (i < 10) {
        print i;

        i = i + 1;
    };

    // 'While' can optionally have an 'else' block, which is taken which the conditional
    // of the while is false. In this case, 'While' can return a value
    var numbers = [5]int{ 1, 33, 7, 99, 64 };
    i = 0;
    var found_five = while (i < numbers.len) {
        if (numbers[i] == 5) {
            break true;
        };

        i = i + 1;
    } else {
        false
    };
    print found_five;

    // While can 'continue'
    i = 0;
    while (i <= 10) {
        i = i + 1;

        if (i >= 3 and i <= 7) {
            continue;
        };

        print i;
    };

    // Blocks can also return values
    var accumulate = {
        var count = 0;
        var j = 0;
        while (j < 10) {
            count = count + j;
            j = j + 1;
        };
        count
    };
}

// The final expression in a function is returned
fn sum(a: int, b: int) int {
    // You can also return early
    if (a < 0) {
        print "a may not be negative".ptr;
        return 0;
    };

    // You can also do 'return a + b;` as the last statement
    // Note that if we wrote 'a + b;' that would be a statement which would return 'void'
    a + b
}
```

## Types
```zig
// User types
struct PrimitiveTypes {
    num: int, // i64
    ch: byte, // u8
    ok: bool, // i64
    char: byte,
};

fn main() void {
    // structs
    var user_struct = PrimitiveTypes{
        .num = 666,
        .ch = 77b, // byte literal
        .ok = true,
        .char = 'a', // char literal (type is byte/u8)
    };
    print user_struct.num; // read member
    user_struct.ok = false; // set member

    // pointers
    var num = 55;
    var p_num = &num; // get address of var
    print 5 + (deref p_num); // deref pointer;
    deref p_num = 7;
    print (deref p_num);
    p_num = &user_struct.num;

    // arrays (treat these as values - they are not a pointers)
    var my_array = [3]int{ 9898, 8, 22 }; // constructing arrays
    print my_array[2]; // array indexing (no bounds checking)
    print my_array.len; // length access
    my_array.ptr; // get pointer to first element. Elements are contiguous
    var string_1 = "Hello world!"; // String literals are array types. There is an implicit
                                   // null terminator that does not count to the length
                                   // of the string. This type is '[12]byte'
    var string_2 = "wow!"; // Note that string_1 and string_2 have different types
    var my_copy = my_array; // this does a full copy

    // slices (treat these as values: a pointer and a length bundled together)
    var slice_1 = @ptrToSlice(string_1.ptr, string_1.len);
    var slice_2 = @ptrToSlice(string_2.ptr, string_2.len);
    // note that slice_1 and slice_2 have the same type: '[]byte'
    print slice_1.len == slice_2.len; // length access
    print slice_1.ptr; // pointer to first element
    slice_2[1]; // slice indexing (no bounds checking)

    // 'anyopaque' is a special type used for type erasure. Helps support FFI
    // for things like printf. Can only pass things that would fit in a register
    // (e.g. int, pointers)
}
```

## Builtins

```zig
import cstd "cstd.code";

struct UserStruct {
    num: int, // i64
    ch: byte, // u8
    ok: bool, // i64
    char: byte,
};

fn main() void {
    // Builtins are function defined by the compiler and usually make use of information
    // only available to the compiler (e.g. types)

    print @sizeOf(byte); // size of type in bytes
    'c' == @intCast(byte, 55); // cast between int types, as there is no 
                               // implicit casting (also not safety checks)

    // aribrarily change pointer types
    var user = @ptrCast(*UserStruct, cstd.malloc(@sizeOf(UserStruct))); 
    
    // convert pointer to int
    if (@ptrToInt(user) == 0) {
        print "Malloc failed".ptr;
    };

    // convert int to pointer
    var p_user = @intToPtr(*UserStruct, 0);

    // create slices
    var my_slice = @ptrToSlice(@ptrCast(*int, cstd.malloc(@sizeOf(int) * 55)), 55);

    print @srcFile(); // the name of the current source file (type is '*byte' with null terminator)
    print @srcLine(); // the line number this was called on
}
```

## FFI

```zig
// Linux (c) abi support is very limited. Only things that are passed via registers are supported
// https://wiki.osdev.org/System_V_ABI#x86-64

// Variadic funcions are not suppored. They must have a fixed number of args when defined
foreign fn printf(format: *byte, data: anyopaque) void;
foreign fn fprintf(stream: *anyopaque, format: *byte, data1: anyopaque, data2: anyopaque, data3: anyopaque)

// c_int can only be used in an FFI context and is an i32
// additionally c_bool is the same, but is a u8 type
foreign fn fseek(stream: *anyopaque, offset: c_int, whence: c_int) c_int;

// raylib examples
// https://www.raylib.com/cheatsheet/cheatsheet.html
foreign fn InitWindow(width: c_int, height: c_int, title: *byte) void;
foreign fn WindowShouldClose() c_bool;
foreign fn BeginDrawing() void;
foreign fn EndDrawing() void;
foreign fn CloseWindow() void;
foreign fn ClearBackground(color: c_int) void;
foreign fn DrawText(text: *byte, pos_x: c_int, pos_y: c_int, font_size: c_int, color: c_int) void;

// argc and argv are implicitly passed to main (in that order), 
// and if main returns an int will use that as a return value
fn main(argc: int, argv: **byte) int { 0 }

// the following are okay
fn main() int { ... }
fn main(argc: int) void { ... }

// The following is not okay
fn main(argv: **byte) void { ... }
```
