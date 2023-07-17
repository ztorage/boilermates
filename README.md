## `boilermates` – A boilerplate generator for similar `struct`s

*It's like "boilerplates", but they're m... mates... G... Get it??*

### What this is not

It's not an attempt at inheritance for Rust.

### Ok, what is it then?

It's a proc_macro for generating:
- Similar structs that have some of the same fields
- Implementations for easily converting between them (with `From`/`Into` where possible, and other methods where not)
- Traits to identify types with common fields so that it's possible to implement functions that require certain fields once for all types who have them.

### Why?

It's a story as old as time. You're implementing an API, you have your input type, your output type, and your internal type that presumably goes to a DB. They're mostly the same, with an `id`, a `checksum`, or some private data sprinkled here and there.

And so you end up with either one `struct` that has many `Option`s, or multiple `struct`s with many conversion implmentations between them. And if they have common functionality between them, with the same implementation, because it uses the same fields, you need to copy-paste it around.

Yes, it's not *that* bad, but the code ends up messy, just because you don't the object's ID when the user sends it in to be created, etc.

### How it works

Take for example the following code:

```rust
use boilermates::boilermates;
use serde::{Deserialize, Serialize};
use uuid::Uuid;

// This is for illustration purposes
const UNIT_PRICE: u64 = 100;
const SHIPPING_PRICE: u64 = 50;

#[boilermates("OrderRequest", "OrderResponse")]
#[boilermates(attr_for("OrderRequest", "#[derive(Clone, Debug, Deserialize)]"))]
#[boilermates(attr_for("OrderResponse", "#[derive(Clone, Debug, Serialize)]"))]
#[derive(Clone, Debug, Deserialize, Serialize)]
struct Order {
    user_id: u64,
    amount: u64,
    address: String,
    #[serde(default)]
    comments: Option<String>,
    shipping_required: bool,

    #[boilermates(not_in("OrderRequest"))]
    id: Uuid,
    
    #[boilermates(only_in("OrderRequest"))]
    jwt_token: String,

    #[boilermates(only_in("Order", "OrderResponse"))]
    #[boilermates(default)]
    status: OrderStatus,

    #[boilermates(only_in_self)]
    #[boilermates(default)]
    assigned_employee_id: Option<u64>,

    #[boilermates(only_in("OrderResponse"))]
    #[boilermates(default)]
    response_code: ResponseCode,
}

// This is for illustration purposes
#[derive(Clone, Debug, Deserialize, Serialize)]
enum OrderStatus {
    Received,
    Packaging,
    Shipped,
}

impl Default for OrderStatus {
    fn default() -> Self {
        Self::Received
    }
}

// This is for illustration purposes
#[derive(Clone, Debug, Deserialize, Serialize)]
enum ResponseCode {
    Ok,
    BadRequest,
    ServerError,
}

impl Default for ResponseCode {
    fn default() -> Self {
        Self::Ok
    }
}
```

#### Struct generation

This will create 3 structs.

First, `Orders`, will have all of the fields, except for `jwt_token`, which only exists in `OrderRequest` since `#[boilermates(only_in("OrderRequest"))]`, and `response_code` which only exists in `OrderResponse` because of `#[boilermates(only_in("OrderResponse"))]`.
```rust
struct Order {
    user_id: u64,
    amount: u64,
    address: String,
    #[serde(default)]
    comments: Option<String>,
    shipping_required: bool,
    status: OrderStatus,
    id: Uuid,
    assigned_employee_id: Option<u64>,
}
```

Then, `OrderRequest`, which won't have `id` since it's marked `#[boilermates(not_in("OrderRequest"))]`, `status` since it's not mentioned in `#[boilermates(only_in("Order", "OrderResponse"))]`, and `assigned_employee_id` because it's marked `#[boilermates(only_in_self)]`, which is synonymous with `#[boilermates(only_in("Order"))]`. It will however, have `jwt_token`:
```rust
struct OrderRequest {
    user_id: u64,
    amount: u64,
    address: String,
    #[serde(default)]
    comments: Option<String>,
    shipping_required: bool,
    jwt_token: String,
}
```

And finally, `OrderResponse`, which will have everything `Order` has plus the `response_code` field, but not `assigned_employee_id`, which is frankly none of the customer's business:
```rust
struct OrderResponse {
    user_id: u64,
    amount: u64,
    address: String,
    #[serde(default)]
    comments: Option<String>,
    shipping_required: bool,
    id: Uuid,
    status: OrderStatus,
    response_code: ResponseCode,
}
```

#### Conversion

Now for the fun stuff. Let's say we've received a new order through the API, and we have an `OrderRequest` in the `request` variable. We can easily convert it to an `Order`, only filling in the missing data. We can do it in two ways. First, use the `into_order` method, which takes the arguments missing in `Order` in the order in which they're written in the original `struct` declaration. Its signature is `pub fn into_order(self, id: Uuid, status: OrderStatus assigned_employee_id: Option<u64> ) -> Order`, so we can do this:
```rust
let order = request.into_order(Uuid::new_v4(), OrderStatus::Received, None);
```

But, `status` and `assigned_employee_id` are marked as `#[boilermates(default)]`, so if we want to use the default values when converting to a type that has these fields, we can use:
```rust
let order = request.into_order_defaults(Uuid::new_v4);
```

Next, after we've successfully saved the order in the DB, we can convert it `OrderResponse` like so:
```rust
let response = order.into_order_response(ResponseCode::Ok);
```

But, since `ResponseCode` has a `Default` implementation, `return_code` is marked `#[boilermates(default)]`, and all other fields from `Order` are present in `OrderResponse`, we can do:
```rust
let response = OrderResponse::from(order); // or `let response: OrderResponse = order.into()`
```

The `From`/`Into` conversion is implemented in all cases when conversion is possible without additional arguments.

#### Blanket implementations

Each field triggers the generation of a `Has{Field}` trait with a getter method `fn {field}(&self) -> &{field_type}` with an implementation for each type that has field.

Since the 3 structs share the much of the same data, they can implement some of the same functionality. For instance, if we'd like to find out what's the order total (remember `UNIT_PRICE` and `SHIPPING_PRICE` in the beginning of the example?), we can create a blanket implementation using the `HasAmount` and `HasShippingRequired` traits, which are implemented for all types that have the `amount` and `shipping_required` fields. It allows us to use the `amount()` and `shipping_required()` getter methods like so:
```rust
trait GetTotal: HasAmount + HasShippingRequired {
    fn total(&self) -> u64 {
        self.amount() * UNIT_PRICE
            + if *self.shipping_required() {
                SHIPPING_PRICE
            } else {
                0
            }
    }
}
impl<T: HasAmount + HasShippingRequired> GetTotal for T {}

// Now all of these work:
let total = request.total()
let total = order.total()
let total = response.total()
```

In a similar fashion, a 'HasNo{Field}' trait  is generated for each struct that does not contain a specific field.
