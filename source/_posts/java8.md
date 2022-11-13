---
title: Dynamic Tests in JUnit5 | Advance Tests in JUnit 5
---

###  Dynamic Tests in JUnit5 | Advance Tests in JUnit 5

Probably you've worked with Juniper. Regularly we annotate @Test  over methods we want to specify as test methods.

These test cases are static in the sense that they are fully specified at compile time, and their behavior cannot be changed by anything happening at runtime. Assumptions provide a basic form of dynamic behavior but are intentionally rather limited in their expressiveness.

In addition to these standard tests a completely new kind of test programming model has been introduced in JUnit 5 Jupiter. This new kind of test is a dynamic test which is generated at runtime by a factory method that is annotated with @TestFactory.

A DynamicTest is a test case generated at runtime. It is composed of a display name and an Executable. Executable is a @FunctionalInterface which means that the implementations of dynamic tests can be provided as lambda expressions or method references.

For example, I want to test my shopping cart service works fine, when 20 requests for adding Items to card being sent, I want to verify that using all the requests processed for example items saved into database successfully ( test scenario1), then processed items are sent into Kafka topic (test scenario2).


### Example processing add items to shopping card scenario


```java
    shoppingCartService.addProductToShoppingCart(shoppingCart.getUserId(), shoppingCart.getAsin());
    assertFalse(shoppingCartService.getProductsInCartByUserId(shoppingCart.getUserId()).isEmpty());
    verify(shoppingCartRepository).save(shoppingCart);
```