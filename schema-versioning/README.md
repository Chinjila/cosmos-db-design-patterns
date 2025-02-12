---
page_type: sample
languages:
- csharp
products:
- azure-cosmos-db
name: |
  Azure Cosmos DB design pattern: Schema Versioning
urlFragment: schema-versioning
description: This example demonstrates how to implement a schema versioning making it possible to track the changes of schema through a schema tracking field.
---

# Azure Cosmos DB design pattern: Schema versioning

Schema versioning is used to track the schema changes of a document. The schema version can be tracked in a field in the document, such as `SchemaVersion`. If a document does not have the field present, it could be assumed to be the original version of the document.

Schema versioning allows NoSQL databases to evolve with applications, minimizing disruptions and preserving data integrity. However, managing schema versioning requires a clear strategy and processes to handle changes effectively.

This sample demonstrates:

- ✅ Using a data generator to generate original carts and schema-versioned carts.
- ✅ Running a website to show data generated in Azure Cosmos DB.

## Common scenario

As the amount of data grows and the usage of the data grows, it may make sense to restructure the document. Schema versioning makes it possible to track the changes of schema through a schema tracking field. When using schema versioning, it is also advised to keep a file with release notes that explain what changes were made in each version.

## Sample implementation of schema versioning

Suppose the Wide World Importers had an online store with data in Azure Cosmos DB for NoSQL. This is the initial cart object.

```csharp
    public class Cart
    {
        [JsonProperty("id")]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        public string SessionId { get; set; } = Guid.NewGuid().ToString();
        public int CustomerId { get; set; }
        public List<CartItem>? Items { get; set;}
    }

    public class CartItem {
        public string ProductName { get; set; } = "";
        public int Quantity { get; set; }
    }
```

When stored in Azure Cosmos DB for NoSQL, a cart would look like this:

```json
{
    "id": "194d7453-d9db-496b-834b-7b2db408e4be",
    "SessionId": "98f5621e-b1af-44f1-815c-f4aac728c4d4",
    "CustomerId": 741,
    "Items": [
        {
            "ProductName": "Product 23",
            "Quantity": 4
        },
        {
            "ProductName": "Product 16",
            "Quantity": 3
        }
    ]
}
```

This model was initially designed assuming products were ordered as-is without customizations. However, after feedback, they realized they needed to track special order details. It does not make sense to update all cart items with this feature, so adding a schema version field to the cart can be used to distinguish schema changes. The changes in the document would be handled at the application level.

This could be the updated class with schema versioning:

```csharp
    public class CartWithVersion
    {
        [JsonProperty("id")]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        public string SessionId { get; set; } = Guid.NewGuid().ToString();
        public long CustomerId { get; set; }
        public List<CartItemWithSpecialOrder>? Items { get; set;}
        // Track the schema version
        public int SchemaVersion = 2;
    }

    public class CartItemWithSpecialOrder : CartItem {
        public bool IsSpecialOrder { get; set; } = false;
        public string? SpecialOrderNotes {  get; set; }
    }
```

An updated cart in Azure Cosmos DB for NoSQL would look like this:

```json
{
    "SchemaVersion": 2,
    "id": "9baf08d2-e119-46a1-92d7-d94ee59d7270",
    "SessionId": "39306d1b-d8d8-424a-aa8b-800df123cb3c",
    "CustomerId": 827,
    "Items": [
        {
            "IsSpecialOrder": false,
            "SpecialOrderNotes": null,
            "ProductName": "Product 4",
            "Quantity": 2
        },
        {
            "IsSpecialOrder": true,
            "SpecialOrderNotes": "Special Order Details for Product 22",
            "ProductName": "Product 22",
            "Quantity": 2
        },
        {
            "IsSpecialOrder": true,
            "SpecialOrderNotes": "Special Order Details for Product 15",
            "ProductName": "Product 15",
            "Quantity": 3
        }
    ]
}
```

When it comes to data modeling, a schema version field in a JSON document can be incremented when schema changes happen. This could be used if data modeling happens with one team while development is handled separately. They could have a schema version document to help track these changes. In this example, it could look something like this:

---
**Filename**: schema.md

**Schema Updates**

| Version | Notes |
|---------|-------|
| 2 | Added special order details to cart items |
| (null) | original release |

---

If you use a nullable type for the version, this will allow the developers to check for the presence of a value and act accordingly.

In [the demo](./source/setup.md), `SchemaVersion` is treated as a nullable integer with the `int?` data type. The developers added a `HasSpecialOrders()` method to help determine whether to show the special order details. This is what the Cart class looks like on the website side:

```csharp
    public class Cart
    {
        [JsonProperty("id")]
        public string Id { get; set; } = Guid.NewGuid().ToString();
        public string SessionId { get; set; } = Guid.NewGuid().ToString();
        public long CustomerId { get; set; }
        public List<CartItemWithSpecialOrder>? Items { get; set;}
        public int? SchemaVersion {get; set;}
        public bool HasSpecialOrders() { 
            return this.Items.Where(x=>x.IsSpecialOrder == true).Count() > 0;
        }
    }
```

The website demo shows the output based on conditional handling.

```razor
@foreach (Cart cart in Model.Carts){
    <section data-id="@cart.Id">
        <p><strong>Customer </strong>@cart.CustomerId</p>
        <table>
            <thead>
                <tr>
                    @if(cart.SchemaVersion != null){
                        <th>Schema Version</th>
                    }                        
                    <th>Product Name</th>
                    <th>Quantity</th>
                    @if (cart.HasSpecialOrders()){
                        <th>Special Order Notes</th>
                    }
                </tr>
            </thead>
        @foreach (var item in cart.Items)
        {
            <tr>
                @if(cart.SchemaVersion != null){
                    <td>@cart.SchemaVersion</td>
                }
                <td>@item.ProductName</td>
                <td>@item.Quantity</td>
                @if (cart.HasSpecialOrders()){
                    <td>
                    @if (item.IsSpecialOrder){
                        @item.SpecialOrderNotes
                    }
                    </td>
                }
            </tr>
        }
        </table>
    </section>
}
```

When you need to keep track of schema changes, use this schema versioning pattern.

## Try this implementation

In order to run the demos, you will need:

- [.NET 6.0 Runtime](https://dotnet.microsoft.com/download/dotnet/6.0)
- [Azure Functions Core Tools v4](https://learn.microsoft.com/azure/azure-functions/functions-run-local#install-the-azure-functions-core-tools)

## Confirm required tools are installed

Confirm you have the required versions of the tools installed for this demo.

First, check the .NET runtime with this command:

```bash
dotnet --list-runtimes
```

As you may have multiple versions of the runtime installed, make sure that .NET components with versions that start with 6.0 appear as part of the output.

Next, check the version of Azure Functions Core Tools with this command:

```bash
func --version
```

You should have installed a version that starts with `4.`. If you do not have a v4 version installed, you will need to uninstall the older version and follow [these instructions for installing Azure Functions Core Tools](https://learn.microsoft.com/azure/azure-functions/functions-run-local#install-the-azure-functions-core-tools).

## Getting the code

### **Clone the Repository to Your Local Computer:**

**Using the Terminal:**

- Open the terminal on your computer.
- Navigate to the directory where you want to clone the repository.
- Type `git clone git clone https://github.com/Azure-Samples/cosmos-db-design-patterns.git` and press enter.
- The repository will be cloned to your local machine.

**Using Visual Studio Code:**

- Open Visual Studio Code.
- Click on the **Source Control** icon in the left sidebar.
- Click on the **Clone Repository** button at the top of the Source Control panel.
- Paste `https://github.com/Azure-Samples/cosmos-db-design-patterns.git` into the text field and press enter.
- Select a directory where you want to clone the repository.
- The repository will be cloned to your local machine.

### **GitHub Codespaces**

You can try out this implementation by running the code in [GitHub Codespaces](https://docs.github.com/codespaces/overview)

- Open the application code in a GitHub Codespace:

    [![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/azure-samples/cosmos-db-design-patterns?quickstart=1&devcontainer_path=.devcontainer%2Fschema-versioning%2Fdevcontainer.json)

## Create an Azure Cosmos DB for NoSQL account

1. Create a free Azure Cosmos DB for NoSQL account: (<https://cosmos.azure.com/try>)

1. In the Data Explorer, create a new database and container with the following values:

    | | Value |
    | --- | --- |
    | **Database name** | `Sales` |
    | **Container name** | `Carts` |
    | **Partition key path** | `/id` |
    | **Throughput** | `1000` (*Autoscale*) |

**Note:** We are using shared database throughput because it can scale down to 100 RU/s when not running. This is the most cost effient if running in a paid subscription and not using Free Tier.

## Set up environment variables

You need 2 environment variables to run these demos.

1. Go to resource group.

1. Select the new Azure Cosmos DB for NoSQL account.

1. From the navigation, under **Settings**, select **Keys**. The values you need for the environment variables for the demo are here.

Create 2 environment variables to run the demos:

- `COSMOS_ENDPOINT`: set to the `URI` value on the Azure Cosmos DB account Keys blade.
- `COSMOS_KEY`: set to the Read-Write `PRIMARY KEY` for the Azure Cosmos DB for NoSQL account

Create your environment variables in a bash terminal with the following syntax:

```bash
export COSMOS_ENDPOINT="YOUR_COSMOS_ENDPOINT"
export COSMOS_KEY="YOUR_COSMOS_KEY"
```

## Generate data

Open the application code. Run the data generator to generate original carts and schema-versioned carts.

```bash
cd ./data-generator
dotnet run
```

The number of carts that you specify will be doubled. The generator generates the same number of original carts and versioned carts.

The output will look something like this:

```bash
This code will generate sample carts and create them in an Azure Cosmos DB for NoSQL account.
The primary key for this container will be /id.


Enter the database name [default:CartsDemo]:

Enter the container name [default:Carts]:

How many carts should be created?
3
Check Carts for new carts
Press Enter to exit.
```

## Run the website to show generated data

Run the website to display the carts.

```bash
cd ./website
dotnet run
```

Navigate to the URL displayed in the output. In the example below, the URL is shown as part of the `info` output, following the "Now listening on:" text.

![Screenshot of the 'dotnet run' output. The URL to navigate to is highlighted. In the screenshot, the URL is 'http://localhost:5279'.](./images/local-site-url.png)

The output will show a variety of randomly generated carts and include the schema version when populated. When a cart contains no special items, the Special Order Notes field will not appear in the cart table.

![Screenshot of the schema-versioned carts demo. The first cart shows 2 items with the fields Product Name and Quantity. The second cart shows 3 items with the fields for Schema Version, Product Name, Quantity, and Special Order Notes. The third cart shows 1 item with the fields for Schema Version, Product Name, Quantity, and Special Order Notes. The fourth cart shows 1 item with the fields for Schema Version, Product Name, and Quantity.](./images/schema-versioned-carts-website.png)

## Summary

Schema versioning is a valuable design pattern within Azure Cosmos DB due to its NoSQL nature and the dynamic requirements of modern applications. In the context of Azure Cosmos DB:

- **Flexibility**: Azure Cosmos DB's schema-less nature aligns well with schema versioning. As applications evolve, schema changes can be seamlessly introduced without disrupting existing data.

- **Adaptability**: With multiple API options like SQL, MongoDB, Cassandra, and Gremlin, Azure Cosmos DB caters to diverse data needs. Schema versioning ensures smooth transitions across APIs and helps maintain consistency.

- **Continuous Development**: Azure Cosmos DB is a cornerstone of cloud-native applications that demand rapid iteration. Schema versioning empowers continuous development by accommodating changing data structures while upholding data integrity.

- **Backward Compatibility**: The ability to coexist with multiple schema versions guarantees backward compatibility. Applications can function with data in both old and new formats during the transition period.

- **Data Security**: By minimizing abrupt schema changes, schema versioning reduces the risk of data loss and corruption. It enables gradual and controlled adaptation to evolving requirements.

In summary, schema versioning is a crucial design pattern for Azure Cosmos DB, promoting agility, compatibility, and robust data management within the dynamic landscape of modern applications.
