Rhetos CommonConcepts package contains some simple business rules that are common in many applications.
Then can often be used by simply declaring them on an entity or a property.

Contents:

1. [Property value constraints](#property-value-constraints)
2. [Deny data modifications](#deny-data-modifications)
3. [Automatically generated data](#automatically-generated-data)
4. [Logging data changes and auditing](#logging-data-changes-and-auditing)
5. [Other features](#other-features)
6. [See also](#see-also)

## Property value constraints

Property value constraints are simple business rules that can be declared beside the property in the .rhe script.
The following concepts are available in CommonConcepts package:

* `Required` - The property value must be entered when saving a record.
  There are two subvariant of this concept:
  * `SystemRequired` - The user does not need to enter the property value, but its value should be entered (perhaps automatically by some other business rules).
  * `UserRequired` - User (or client applications) needs to provide the property value.
  * Note: The required data validation is implemented in the application layer,
    without the **NOT NULL** constraint in database.
    A required property has nullable type in both C# and database.
    This design decision simplifies interaction between database changes by Rhetos and custom data-migration scripts.
    It also makes the deployment process more robust and allows more flexibility when modifying business rules
    (with focus on maintainability of large business applications).
* `Unique` - Two records cannot have same value of this property.
  * `UniqueMultiple` - A unique constraint over multiple properties:
    Two records cannot have same combination of values.
* `MinValue` - Limit the smallest allowed value of the property.
* `MaxValue` - Limit the largest allowed value of the property.
* `MinLength` - Limit the smallest allowed length of the string.
* `MaxLength` - Limit the largest allowed length of the string.
* `RegExMatch` - Use a regular expression to validate the string property value.
  The validation is checked only if the property value is entered (add Required concept if needed).
  The provided pattern must match the *whole* property value (`^` and `$` are automatically added to pattern).

For more complex data validations a specific filter can be developed,
see [Data validations](Data-validations) article for more info.
Such validations are built in two steps:

1. Create a filter on the entity.
   In the example below, the filter FinishBeforeStart returns every Employee with invalid data
   (WorkFinished is before WorkStarted).
2. Create a validation rule for that filter, with the message for the user.
   The validation will return the error message it the user tries to enter the invalid data.

```c
Entity Employee
{
    Integer IdentificationNumber;
    ShortString LastName { Required; }
    ShortString FirstName { Required; }
    ShortString Code { RegExMatch "\d{7,10}" "Code must have 7 to 10 digits."; }
    DateTime WorkStarted { Required; }
    DateTime WorkFinished;
    Integer TestPeriod { MinValue 1; MaxValue 12; }
    ShortString Iban { Required; Unique; MinLength 21; MaxLength 21; }

    UniqueMultiple 'LastName FirstName';

    ItemFilter FinishBeforeStart 'employee => employee.WorkFinished != null
        && employee.WorkFinished.Value < employee.WorkStarted.Value';
    InvalidData FinishBeforeStart 'It is not allowed to enter a WorkFinished time before the WorkStarted time.';
}
```

## Deny data modifications

The following business rules are similar to data validations, but actually act more like action permissions.
Instead of checking if a certain data value *is valid or not*, they just block certain operations depending on the context.
Examples of the following concepts are available in a unit testing DSL script
[Validations.rhe](https://github.com/Rhetos/Rhetos/blob/master/test/CommonConcepts.TestApp.Shared/DslScripts/Validations.rhe).

* `Lock` - Deny update and delete of the entity records, for records in a certain state (provided by a filter).
* `LockProperty` - Deny update of a property, for records in a certain state.
* `LockExcept` - Deny update (except for properties from the provided list) and delete of the entity records, for records in a certain state.
* `DenyUserEdit` on an entity - Client application is not allowed to directly insert, update or delete the entity records (no condition).
* `DenyUserEdit` on a property - Client application is not allowed to directly insert or update the property.
  * When inserting and updating records, the client is only allowed to send the previous value, or a *null* value (it will be interpreted as "no change" on update).
  * Best practices: Instead of using this concept, consider separating the properties computed by the system from properties entered by a user into different entities.

Similar features and alternatives:

1. If some data operation should be allowed or denied based on *which user* is performing it,
   you should use [Row permissions](RowPermissions-concept) or [Basic permissions](Basic-permissions)
   instead of the concepts listed above.
2. If you need to implement a very specific business rule for which there is no
   standard concept available
   (for example to deny deleting certain records while allowing updates),
   it would be best to create [your own concept](Rhetos-concept-development)
   based on the implementation of a similar one.
   A more pragmatic option is to directly insert an arbitrary C# code into the entity's
   Save method with [Low-level object model concepts](Low-level-object-model-concepts).

## Automatically generated data

* `DefaultValue` - Generate the default property value when inserting a new record (since Rhetos v2.10).
* `AutoCode` - Automatically generate numeric codes for each new record.
  * When inserting a new record, the initially entered value is interpreted as a format for generating the code.
    It supports the following formatting:
    * Multiple digits with leading zeros: for example, if "+++" is entered,
      a three-digit number will be created (001, 002, 003, ...).
      After all values have been taken, it will automatically increase the number of digits.
    * Custom prefix: for example, if "BOOK+" is entered,
      the generated codes will be BOOK1, BOOK2, BOOK3, ....
  * `AutoCode`, `DefaultValue` and `DenyUserEdit` are often used together on a same property:
    * Use `DenyUserEdit` if the user should not enter its own code or code format.
      By default, the users can enter the code manually, instead of generating it automatically,
      by entering the value that does not end with "+".
    * Use `DefaultValue` to specify custom prefix and number of digits, if the codes are not simple numbers (1, 2, 3, ...).
  * AutoCode supports `Integer` and `ShortString` property types.
  * AutoCode automatically generates `Unique` index and `SystemRequired` on the property, no need to add them in DSL script.
* `AutoCodeForEach` - Same as `AutoCode`, but the numbers are starting from 1 within each group of records. The group is defined by the second property value.
* `CreationTime` - Automatically enters time when the records was created.
* `ModificationTimeOf` - Automatically enters time when some given property was last updated.

Code examples for **DefaultValue** are available in DSL script [DefaultValueTest.rhe](https://github.com/Rhetos/Rhetos/blob/master/test/CommonConcepts.TestApp.Shared/DslScripts/DefaultValueTest.rhe) script.

Code examples for **AutoCode**, **CreationTime** and **ModificationTimeOf** are available in DSL script [SimpleBusinessLogic.rhe](https://github.com/Rhetos/Rhetos/blob/master/test/CommonConcepts.TestApp.Shared/DslScripts/SimpleBusinessLogic.rhe) script.

In the following example, the book codes are generated automatically, using a 3-digit code format
with the prefix "BOOK" (BOOK001, BOOK002, etc.), and the user cannot enter or change the codes. 

```c
Entity Book
{
    ShortString Code { AutoCode; DefaultValue 'item => "BOOK+++"'; DenyUserEdit; }
}
```

## Logging data changes and auditing

* `Logging` - Creates a database trigger that monitors all inserts, updates and deletes, and writes them to `Common.Log` table.
* `Log` - Extends the trigger to track modified or deleted values for a given property.
* `AllProperties` - Extends the trigger to track modified or deleted values for all entity's properties.

Examples:

```C
Module Demo
{
    Entity Person
    {
        ShortString FirstName;
        ShortString LastName;
        LongString Description;

        Logging
        {
            Log Demo.Person.FirstName;
            Log Demo.Person.LastName;
            // 'Description' property is long and not interesting, so we don't want to log its values.
        }
    }

    Entity Animal
    {
        ShortString Name;
        Decimal MaxSpeed;

        Logging { AllProperties; }
    }
}
```

See [Logging data changes and auditing](Logging#logging-data-changes-and-auditing) for more information.

## Other features

* `Deactivatable` - Allows tracking of active and deactivated records.
  * Internally, it adds a `Bool Active` property to the entity with the default value True.
  * It creates a composable filter `Rhetos.Dom.DefaultConcepts.ActiveItems`,
    with an optional parameter `ItemID`.
    It returns all active items and additionally the item with the given ID.
    This is a common pattern for lookup that needs to display the current item whether it is active or not.
  * Code example is available in DSL script [SimpleBusinessLogic.rhe](https://github.com/Rhetos/Rhetos/blob/master/test/CommonConcepts.TestApp.Shared/DslScripts/SimpleBusinessLogic.rhe).

* `PessimisticLocking` - Enables automatic verification of explicit client locks when saving a record. A user can change a records only when there is no ExclusiveLock from another user on this record. When editing detail records, only master needs to be locked.
  * A client application can manage the locks with actions Common.SetLock and Common.ReleaseLock, with parameters ResourceType (full entity name) and ResourceID (GUID).
  * Code example is available in DSL script [PessimisticLocking.rhe](https://github.com/Rhetos/Rhetos/blob/master/test/CommonConcepts.TestApp.Shared/DslScripts/PessimisticLocking.rhe).

## See also

Examples of many DSL concepts are available in the unit testing setup DSL script in the
Rhetos framework source folder [CommonConcepts\CommonConceptsTest\DslScripts](https://github.com/Rhetos/Rhetos/tree/master/test/CommonConcepts.TestApp.Shared/DslScripts).
