swxben.dataaccess
=================

Simple data access layer for SQL Server


## Installation

Install the engine via [NuGet](http://nuget.org/packages/swxben.dataaccess), either in Visual Studio (right-click project, Manage NuGet Packages, search for docxtemplateengine) or via the package manager console using `Install-Package swxben.dataaccess`.


## Usage

Create a new `DataAccess` instance and pass in the database connection string (do this with your favourite IoC container):

    IDataAccess dataAccess = new DataAccess(@"Server=.\sqlexpress; Database=swxben_dataaccess; User Id=sa; Password=test;");


### Insert

Insert using strongly typed DTOs. The table name is by convention the type name plus `s`.

    dataAccess.Insert(new MyThing { MyId = 13, MyName = "Ben" });

This roughly corresponds to:

    INSERT INTO MyThings(MyId, MyName) VALUES(13, 'Ben')

It actually uses named fields so SQL injection shouldn't be possible in normal use.


### Update

Update needs the field name for the identifier, rather than using a strict convention:

    dataAccess.Update(new MyThing { MyId = 13, MyName = "Ben"}, "MyId");

    UPDATE MyThings SET MyName = 'Ben' WHERE MyId = 13

Compound keys can be specified:

    dataAccess.Update(
        new MyThing { ForeignKey1 = 5, ForeignKey2 = 8, AtDate = DateTime.Now }, 
        "ForeignKey1",
        "ForeignKey2");

        UPDATE MyThings SET AtDate = '...' WHERE ForeignKey1 = 5 AND ForeignKey2 = 8

Alternatively, identifier fields can be decorated:

    class MyThing {
        [DataAccess.Identifier]
        public int MyId;
        public string MyName;
    }

    dataAccess.Update(new MyThing { MyId = 4, MyName = "Ben" });

    UPDATE MyThings SET MyName = 'Ben' WHERE MyId = 4

Multiple fields can be decorated with the `DataAccess.Identifier` attribute for compound keys. Specifying the identifier columns using the first overload above will override the decorated fields.


### Select

The select method uses a strongly typed DTO:

	var myThings = dataAccess.Select<MyThing>(
		where: new { MyName = "Ben" },
		orderBy: "MyId"
	);

    SELECT * FROM MyThings WHERE MyName = 'Ben' ORDER BY MyId


### Execute arbitrary queries

There are two versions of `ExecuteQuery()`, one returns strongly-typed DTOs, the other returns dynamics:

	IEnumerable<MyThing> myThings = dataAccess.ExecuteQuery<MyThing>(
		"SELECT * FROM MyThings WHERE MyName = @MyName", 
		new { MyName = "Ben" });

	IEnumerable<dynamic> myThings = dataAccess.ExecuteQuery<MyThing>(
		"SELECT * FROM MyThings WHERE MyName = @MyName", 
		new { MyName = "Ben" });


### Execute an arbitrary command

	dataAccess.ExecuteCommand(
		"DELETE FROM MyThings WHERE MyId = @MyId",
		new { MyId = 17 });


### Ignore fields and properties

Decorate fields and properties with the `[DataAccess.Ignore]` attribute to indicate that they should be ignored by the `Insert` and `Update` commands.

    class MyThing {
    	public string Name{ get; set; }
    	[DataAccess.Ignore]
    	public int WidgetCount { get; set; }
    }

    dataAccess.Insert(new MyThing { Name = "Ben" });

    INSERT INTO MyThings(Name) VALUES('Ben')

Ignored fields and properties are still assigned when a value comes in from a query:

    dataAccess.ExecuteQuery<MyThing>(
    	"SELECT t.*, (SELECT COUNT(*) FROM Widgets w WHERE w.Owner = t.Name) AS WidgetCount FROM MyThings t");

    /* MyThing.Name == "Ben", MyThing.WidgetCount == 12 */


### DTOs without a parameterless constructor

Most simple DTOs will just have a default constructor, however there are some cases where a DTO 
will require a more complex constructor. `ExecuteQuery<T>()` and `Select<T>()` have overloads
that accept a factory method - an object of type `Func<T>`:

    var service = Get.Some<Service>();

    var things = dataAccess.Select<MyThing>(() => new MyThing(service));
    var someThings = dataAccess.ExecuteQuery<MyThing>(
        () => new MyThing(service),
        "SELECT TOP 5 * FROM MyThings"
    );


### DTOs with a complex graph

This could also be used in place of the factory method above for DTOs with a non-parameterless constructor. `ExecuteQuery<T>()` and `Select<T>()` have overloads that accept a tranform method, a `Func<dynamic, T>` that converts the dynamic row into a T:

    class Address { 
        public string Line1; 
        public string Line2; 
    }
    class Customer { 
        public string Name; 
        public Address BillingAddress; 
        public Address ShippingAddress; 
    }

    var customers = dataAccess.Select<Customer>(transform: r => new Customer { 
            Name = (string)r.Name,
            BillingAddress = new Address { Line1 = (string)r.BillingAddressLine1, Line2 = (string)r.BillingAddressLine2 },
            ShippingAddress = new Address { Line1 = (string)r.ShippingAddressLine1, Line2 = (string)r.ShippingAddressLine2 }
        });
        
    // or:
    Customer GetCustomer(dynamic row) { return ... }
    var customers = dataAccess.Select(GetCustomer);


### Other methods

- `DropTable(string tableName)` drops the specified table if it exists. This is useful for automated tests.
- `TestConnection()` returns any exception thrown when opening the connection. If there are no exceptions, null is returned. This is used to test the connection without bombing out the application.
- `Any<T>(object criteria = null)` returns a boolean indicating if there are any rows in the specified table that satisfy the specified criteria.


## Contribute

If you want to contribute to this project, start by forking the repo: <https://github.com/swxben/swxben.dataaccess>. Create an issue if applicable, create a branch in your fork, and create a pull request when it's ready. Thanks!

### Contributors

## Building, running tests and the NuGet package

THe VS2010 solution is in the root folder. Unit tests (src\Swxben.DataAccess.Tests\bin\Debug\Swxben.DataAccess.Tests.dll) can be run in a console using `tests.bat`. The NuGet package can be built by running `build-nuget-package.cmd`. A database named `swxben_dataaccess` running on `.\sqlexpress` is needed.


## Licenses

All files [CC BY-SA 3.0](http://creativecommons.org/licenses/by-sa/3.0/) unless otherwise specified.

### Third party licenses

Third party libraries or resources have been included in this project under their respective licenses.




