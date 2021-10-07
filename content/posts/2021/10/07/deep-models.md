---
title: "Deep Models"
date: 2021-10-07T09:02:50+01:00
draft: false
toc: false
images:
categories:
  - software development
tags:
  - kotlin
---

Modeling is a critical task in Software Development: good models reduce the risk of bugs, increase readability and improve maintainability. However, we can often see developers focusing on *"How can I code this?"*, and they are done with their task when they find a way to code it. This inherently reduces the domain modeling to the abuse of primitives and shallow design where any inconsistent state is allowed.

To better explain, let's consider the hypothetical requirements:
- Your team is developing a feature to order items in e-commerce.
- Your job is to design the models of this feature.

The simplest way to do that might look like the code below.

```kotlin
fun findOrderById(id: String): Order

data class Order(
    val id: String,
    val accountId: String,
    val items: List<OrderItem>,
    // ...billing address, shipping address, status, and other properties.
)

data class OrderItem(
    val id: String,
    val name: String,
    val quantity: Int,
    val price: BigDecimal,
)
```

At first glance, these models will look good, but let's have a closer look.
1. [Primitive Obsession](https://refactoring.guru/smells/primitive-obsession): almost all types are String, and the compiler is incapable of verifying their integrity.
2. [Absence of Invariants](https://medium.com/code-design/invariants-in-code-design-557c7864a047): it is unrealistic to define a quantity as a number between `-2147483648` and `2147483647`.
3. [Error Prone](https://www.youtube.com/watch?v=t3DBzaeid74): what happens if you create an `Order` with an empty `accountId`? What will happen if you call `findOrderById` with an `ItemId` by mistake?

Good models get out of developers' way by making it impossible to be in an invalid state. More than that, it leverages the compiler to not allow human's mistake (e.g., an `OrderItemId` should not be used as an `OrderId`).

Let's redesign the previous model.

```kotlin
fun findOrderById(id: OrderId): Order

data class Order(
    val id: OrderId,
    val accountId: OrderAccountId,
    val items: List<OrderItem>,
)

@JvmInline
value class OrderId(val value: String) {
    init {
        runCatching { UUID.fromString(value) }
            .onFailure { e -> error("OrderId should be a valid UUID but found: $value.") }
    }
}

@JvmInline
value class AccountId(val value: String) {
    init {
        runCatching { UUID.fromString(value) }
            .onFailure { e -> error("AccountId should be a valid UUID but found: $value.") }
    }
}

data class OrderItem(
    val id: OrderItemId,
    val name: OrderItemName,
    val quantity: OrderItemQuantity,
    val price: OrderItemPrice,
)

@JvmInline
value class OrderItemId(val value: String) {
    init {
        runCatching { UUID.fromString(value) }
            .onFailure { e -> error("OrderItemId should be a valid UUID but found: $value.") }
    }
}

@JvmInline
value class OrderItemName(val value: String) {
    init {
        require(value.isNotBlank()) { "OrderItemName should not be blank." }
        require(value.length < 200) { "OrderItemName length should be smaller than 200 but found: ${value.length} with content: $value."}
    }
}

@JvmInline
value class OrderItemQuantity(val value: Int) {
    init {
        require(value < 0) { "OrderItemQuantity should not be negative but found: $value." }
        require(value > 99) { "OrderItemQuantity should not be higher than 99 but found: $value." }
    }
}

@JvmInline
value class OrderItemPrice(val value: BigDecimal) {
    init {
        require(value < 0) { "OrderItemPrice should not be negative but found: $value." }
    }
}
```

The new API makes it impossible to create an invalid instance of an `Order`. It ensures all invariants of an `Order` are respected during the application's lifetime and reduces corner cases by providing a type-safe API: the function `findOrderById` knows that you are calling it with an `OrderId`, it will be a valid `UUID`.

## F.A.Q.

1. What about the number of classes?

Good models often will involve more classes and specific validations - but keep in mind those are already in our code. Remember all those checks to verify if the parameter of `findOrderById` is valid? Or that `InvalidArgumentException` when the expected quantity but received a negative number? Now they are explicitly defined in our domain and can be easily found.

2. What about the number of lines of code?

It is true that now you need to cover more ground, but it is also true that you reduced the number of corner cases. Before, your tests had to cover if the `findOrderByid` parameter is valid. Now, the compiler will ensure the parameter is always correct - if the code compiles, you know you are receiving a correct `UUID`. In the end, you are making safer code and reducing the number of tests you will need.

3. Should I have a class for each property?

Probably not. Modeling requires a lot of analysis. You must identify the essential concepts of your domain that must be assertive and ensure those are deep. For that, [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html) might help you.

# Credits

Special credits for [Secure By Design](https://www.manning.com/books/secure-by-design) (Chapter 2: Shallow Modeling) and [A Philosophy of Software Design](https://www.amazon.de/-/en/John-Ousterhout/dp/1732102201) (Chapter 4: Modules should be deep) where I learnt about Deep and Shallow modeling, both exceptional books.

If you like my posts, follow me on Twitter: [@marcellogalhard](https://twitter.com/marcellogalhard)