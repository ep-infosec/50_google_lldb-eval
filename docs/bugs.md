# Log of mismatches between LLDB and lldb-eval

Categories:

*   [Bugs in lldb-eval](#bugs-in-lldb-eval)
*   [Undefined behaviours](#undefined-behaviours)
*   [Possible bugs in LLDB](#possible-bugs-in-lldb)

## Bugs in lldb-eval

*   Using `nullptr` in boolean context

    ```cpp
    !nullptr
    nullptr ? a : b

    // lldb      : OK
    // lldb-eval : nullptr isn't contextually convertible to bool
    ```

    *   Encountered: 2021 Jan (manual testing)
    *   Fixed in [#51](https://github.com/google/lldb-eval/pull/51)

*   Casting `nullptr_t` to `bool`

    ```cpp
    (bool)nullptr

    // lldb      : false
    // lldb-eval : cast from pointer to smaller 'bool' loses information
    ```

    *   Encountered: 2021 Jan (manual testing)

*   Casting enums to floats

    ```cpp
    > (double)Enum::kOne

    // lldb      : 1.0
    // lldb-eval : 4E-324
    ```

    *   Encountered: 2021 Feb (reported by fuzzer)
    *   Fixed in [#68](https://github.com/google/lldb-eval/pull/68)

*   Taking address of members of invalid class pointers

    ```cpp
    &((TestStruct*) invalid_ptr)->field)

    // lldb      : OK (an address of type `int *`)
    // lldb-eval : no-value, no-error
    ```

    *   Note: In most cases this is equivalent to `invalid_ptr + offset(field)`
        (which doesn't require reading from inaccessible memory). This works in
        LLDB, but it isn't implemented in lldb-eval, because of exceptions where
        offset can't be determined statically, e.g.:
        *   `field` is a reference,
        *   `field` is a virtually inherited field.
    *   Frequently reported by fuzzer

*   Array indirection

    ```cpp
    *array

    // lldb      : <first element>
    // lldb-eval : indirection requires pointer operand ('int [3]' invalid)
    ```

    *   Encountered: 2021 Mar (manual testing)
    *   Fixed in [#80](https://github.com/google/lldb-eval/pull/80)

*   Arrays in boolean context

    ```cpp
    array ? a : b

    // lldb      : OK
    // lldb-eval : 'int [3]' is not contextually convertible to 'bool'
    ```

    *   Encountered: 2021 Mar
    *   Fixed in [#81](https://github.com/google/lldb-eval/pull/81)

*   Enums as array subscript

    ```cpp
    array[an_enum]

    // lldb      : OK
    // lldb-eval : array subscript is not an integer
    ```

    *   Encountered: 2021 Mar (manual testing)
    *   Fixed in [#115](https://github.com/google/lldb-eval/pull/115)

*   Member access on class array

    ```cpp
    struct Foo { int x = 1; };
    Foo foo_array[2];

    > foo_array->x

    // lldb      : 1
    // lldb-eval : member reference type 'TestStruct [2]' is not a pointer;
    //             did you mean to use '.'?
    ```

    *   Note: `foo_array` is contextually converted to pointer to the first
        element of the array. The final result is the `x` field of the first
        element of the array.
    *   Encountered: 2021 Mar (reported by fuzzer)
    *   Fixed in [#116](https://github.com/google/lldb-eval/pull/116)

*   Using compatible array and pointer types in conditional expression

    ```cpp
    condition ? array : pointer

    // lldb      : OK
    // lldb-eval : invalid operands 'int [3]' and 'int *'
    ```

    *   Encountered: 2021 Mar (reported by fuzzer)
    *   Fixed in [#92](https://github.com/google/lldb-eval/pull/92)

*   Arithmetics with array/enum references

    ```cpp
    int array[3] = {1, 2, 3};
    int (&array_ref)[3] = array;  // reference to 'array'

    > array_ref + 1
    // lldb      : OK
    // lldb-eval : invalid operands 'int [3]' and 'int'

    Enum an_enum = Enum::kOne;
    Enum& enum_ref = an_enum;

    > enum_ref - 1
    // lldb      : 0
    // lldb-eval : Assertion failed!
    ```

    *   Reason: Not using dereferenced type in *usual unary conversions*.
        Referenced types didn't pass `Type::IsArrayType()` and
        `Type::IsUnscopedEnum()` checks.
    *   Encountered: 2021 Mar (manual testing)
    *   Fixed in [#90](https://github.com/google/lldb-eval/pull/90)

*   Subscripting `void*`

    ```cpp
    &void_ptr[0]

    // lldb      : error: subscript of pointer to incomplete type 'void'
    // lldb-eval : OK (an address)
    ```

    *   Encountered: 2021 Apr (reported by fuzzer)
    *   Fixed in [#128](https://github.com/google/lldb-eval/pull/128)

*   Casting floats to bool

    ```cpp
    (bool)0.1

    // lldb      : true
    // lldb-eval : false
    ```

    *   Reason: Caused by re-implementation of casting logic in
        [#125](https://github.com/google/lldb-eval/pull/125) to always use types
        from the target.
    *   Encountered: 2021 May (reported by fuzzer)
    *   Fixed in [#133](https://github.com/google/lldb-eval/pull/133)

*   Enumeration comparison

    ```cpp
    enum Enum : int { A = -1, B }

    > Enum::A < Enum::B
    // lldb      : true
    // lldb-eval : false
    ```

    *   Reason: Unsigned values were used to compare enums' underlying values
        `Compare(kind, lhs.GetUInt64(), rhs.GetUInt64())`.
    *   Encountered: 2021 May (reported by fuzzer - enumerations picked up from
        `namespace std` had negative values on Linux)
    *   Fixed in [#134](https://github.com/google/lldb-eval/pull/134)

*   Arithmetic vs. logical shift

    ```cpp
    -4 >> 1

    // lldb      : 2
    // lldb-eval : 2147483646
    ```

    *   Reason: Logical right shift was used for all types. Arithmetic right
        shift was supposed to be used for signed types.
    *   Encountered: 2021 Jun (manual testing)
    *   Fixed in [#145](https://github.com/google/lldb-eval/pull/145)

*   Type promotion in shift operation

    ```cpp
    -1 << 1U

    // lldb      : -2         (int)
    // lldb-eval : 4294967294 (unsigned int)
    ```

    *   Reason:
        [*Usual arithmetic conversions*](https://eel.is/c++draft/expr.arith.conv)
        were performed to deduce the resulting type and type promotions. This
        holds for most of the binary operations. However, for shifts the result
        type is that of the promoted left operand.
    *   Encountered: 2021 Jun (reported by fuzzer)
    *   Fixed in [#147](https://github.com/google/lldb-eval/pull/147)

## Undefined behaviours

*   Division by zero

    ```cpp
    > 1 / 0
    > 1 % 0
    // lldb      : <random value>
    // lldb-eval : 0
    ```

    *   Prevented by UB detection mechanism in lldb-eval.
    *   Occasionally, LLDB reports `Execution was interrupted, reason: Exception
        0xc0000094 encountered at address 0x19c1f3b0105. The process has been
        returned to the state before expression evaluation.` This process may
        take more than 10 seconds!

*   Invalid memory access

    ```cpp
    > *invalid_ptr
    > invalid_ptr->field
    > &(invalid_ptr)->ref_field
    > &(invalid_ptr)->virtual_field
    ```

    *   The most frequently reported mismatch by fuzzer.
    *   In most cases LLDB outputs `supposed to interpret, but failed:
        Interpreter couldn't read from memory`. It could also results with
        "process being returned to the state before evaluation" which can take
        several seconds.
    *   The first two expressions were not reproducible in fuzzer since
        [#63](https://github.com/google/lldb-eval/pull/63).

*   Casting negative floats to unsigned integers

    ```cpp
    > (unsigned int)-1.5

    // lldb      : 0
    // lldb-eval : 4294967295

    > __log2(-1.5)  // takes unsigned int

    // lldb      : <random-value>  (computes __log2 of a random unsigned int)
    // lldb-eval : 31              (computes __log2 of 4294967295)
    ```

    *   Prevented by UB detection mechanism in lldb-eval.

*   Casting to different pointer type and dereferencing

    ```cpp
    int x = 2;

    > (int) *(bool*) &x

    // lldb      : 2
    // lldb-eval : 1
    ```

    *   Note: the expression violates
        [strict aliasing rule](https://gist.github.com/shafik/848ae25ee209f698763cffee272a58f8).
    *   Encountered: 2021 Feb (reported by fuzzer)

*   Null pointer arithmetic

    ```cpp
    (char*)0 + char_min

    // lldb      : 0x00000000ffffff80
    // lldb-eval : 0xffffffffffffff80
    ```

    *   Note: only integer 0 can be added or subtracted from a null pointer.
    *   Prevented by UB detection mechanism in lldb-eval.
    *   Encountered: 2021 May (reported by fuzzer)

*   Subtracting ill-formed pointers

    ```cpp
    (int*)3 - (int*)(5 + __log2(100))

    // lldb      : -3
    // lldb-eval : -2
    ```

    *   Note: `sizeof(int)` is 4, and difference of `int*` is expected to be a
        multiple of 4.
    *   Encountered: 2021 Jun (reported by fuzzer)

## Possible bugs in LLDB

*   Type name isn't identifiable without explicit use of `struct` or `class`
    keywords

    ```cpp
    // Test is class or struct

    > sizeof(Test)
    // lldb      : 'Test' has unknown type; cast it to its declared type
    // lldb-eval : 24
    ```

    *   Note: This was reproducible only on Windows, when `Test` was `struct` or
        `class` (e.g. there's no problem with `union`) that was defined globally
        under no-namespace and contained at least one non-static member.
    *   Workaround is to explicitly use `struct` or `class` keyword, e.g.
        `sizeof(struct Test)`.
    *   Encountered: 2021 Mar

    Similar example is:

    ```cpp
    > (Test*)ptr
    // lldb      : expression failed to parse, fixed expression suggested:
    //             (struct TestStruct*)0
    ```

*   `this` isn't compatible with its own type

    ```cpp
    // Test binary:
    struct TestStruct { ... };
    TestStruct ts;

    // Using APIs:
    lldb::SBValue ts1 = frame.FindVariable("ts");
    lldb::SBValue ts2 = frame.EvaluateExpression("ts");

    > lldb_eval::EvaluateExpression(ts1, "this == (TestStruct*)this", ...);
    // OK (true)

    > lldb_eval::EvaluateExpression(ts2, "this == (TestStruct*)this", ...);
    // comparison of distinct pointer types ('TestStruct *' and 'TestStruct *')
    ```

    *   TODO: Investigate the exact reason for this bug. This could be related
        to equality operator of `lldb::SBType` not working as expected.
    *   Encountered: 2021 Apr 14 (reported by fuzzer while supporting evaluation
        in value context)
