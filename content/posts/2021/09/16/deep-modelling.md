---
title: "Deep Modelling"
date: 2021-09-16T17:37:41+02:00
draft: true
toc: false
images:
categories:
  - software development
tags:
  - android
  - compose
  - gradle
---

WARNING: THAT IS A DRAFT, PLEASE DO NOT SHARE WITHOUT PERMISSION.

Modelling is critical in Software Development: good models reduce the risk of bugs, increase readability and improve maintainability. Unfortunately, most software have seen in my career relies on shallow modelling and developers quite often underestimate how important modelling can be.

**Heads-up:** this article expects you to be familiar with Dagger, Retrofit and Coroutines.

Let's consider the following hypothetical requirement:
- Your team is developing a feature to order items in an E-Commerce.
- Your job is to design the models of this feature.

The simplest way to do that would look somehow like the following:

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

At first glance these models will look good but they have many issues:
1. Primitive Obsession: almost all types are String and the compiler is not capable to identify them.
2. Absence of Invariants: it is unrealistic to define a quantity as a number between `-2147483648` and `2147483647`.
3. Error Prone: what happen if you create an `Order` with an empty `accountId`? What will happen if you call `findOrderById` with an `ItemId` by mistake?

Good models should help developers to write good code by not allowing itself to be in an invalid state and leveraging the compiler to not allow inconsistencies (a order id should not be switched with an item id, for example).

Let's redesign the previous model:

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
        require(value.isNotBlank()) {
            "OrderItemName should not be blank."
        }
    }
}

@JvmInline
value class OrderItemQuantity(val value: Int) {
    init {
        require(value < 0) { "OrderItemQuantity should not be negative but found: $value." }
        require(value > 99) { "OrderItemQuantity should not be bigger than 99 but found: $value." }
    }
}

@JvmInline
value class OrderItemPrice(val value: BigDecimal) {
    init {
        require(value < 0) { "OrderItemPrice should not be negative but found: $value." }
    }
}
```

The new API makes it impossible to create an invalid instance of an `Order`, it ensures all invariants of an `Order` are respected during the lifetime of the application and reduce corner cases by providing a type-safe API: the function `findOrderById` now knows that if you managed to pass a `OrderId` it will be a valid UUID.

## What about the quantity of classes?

That is a common question. Good models most likely involve more classes and validations - but those are already in our code. Instead of doing checks explicitly every time we call `findOrderById` to ensure it is a valid UUID or by verifying if the quantity is not negative, now that is done within our models.

# Credits

If you like my posts, follow me on Twitter: [@marcellogalhard](https://twitter.com/marcellogalhard)