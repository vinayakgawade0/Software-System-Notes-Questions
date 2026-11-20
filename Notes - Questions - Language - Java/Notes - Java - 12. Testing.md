# Notes - Java - 12. Testing

## 1. Unit Testing & Mocking Strategies (JUnit & Mockito)

### Mockito Stubbing: when-then vs doReturn-when

In modern Java unit testing, Mockito is the primary library used for mocking dependencies, stubbing behavior, and verifying interactions. Mockito offers two primary syntaxes for defining mock return values:
1. Standard syntax: `when(mock.method()).thenReturn(value)`
2. Exception / Spy syntax: `doReturn(value).when(mock).method()`

#### Architectural & Behavioral Comparison

| Dimension | `when().thenReturn()` | `doReturn().when()` |
| :--- | :--- | :--- |
| **Type Safety** | Strictly type-checked at compile time. Return type must match the signature. | Generics/Type-checked weakly (`Object` based). Type mismatches can fail at runtime. |
| **Readability** | Fluent, idiomatic English order ("When method called, then return X"). | Inverted phrasing ("Do return X, when mock called with method"). |
| **Real Method Execution on Spies** | **Calls the real method** once during stub definition unless overridden! | **Bypasses the real method** entirely during stub definition. |
| **Final / Void Methods** | Cannot stub `void` methods (`when(void)` is a compile error). | `doNothing()`, `doThrow()`, `doAnswer()` explicitly support `void` methods. |
| **Consecutive / Throw Stubbing** | Fluent chaining: `thenReturn(v1, v2, v3)`. | `doReturn(v1).doReturn(v2).when(...)`. |

#### The Spy Trap with `when().thenReturn()`

When working with `@Spy` (partial mocks where real methods are called unless stubbed), using `when(spy.method()).thenReturn(...)` executes the real method during the stubbing phase itself. If the real method accesses uninitialized state, performs expensive database queries, or throws an exception, the test will fail during setup.

```java
@ExtendWith(MockitoExtension.class)
class PaymentServiceTest {

    @Spy
    private PaymentGateway paymentGateway = new PaymentGateway();

    @Test
    void testSpyStubbingPitfall() {
        // TRAP: paymentGateway.process() real method is actually invoked!
        // If process() contacts a remote network or throws NullPointerException, setup crashes:
        // when(paymentGateway.process(any())).thenReturn(PaymentResponse.success());

        // SAFE SOLUTION: doReturn bypasses real method execution completely:
        doReturn(PaymentResponse.success()).when(paymentGateway).process(any());
        
        PaymentResponse response = paymentGateway.process(new PaymentRequest(100));
        assertThat(response.isSuccess()).isTrue();
    }
}
```

#### Rule of Thumb for Mockito Stubbing
- Use `when(mock.method()).thenReturn(value)` by default for pure `@Mock` instances due to compile-time type safety and fluent readability.
- Use `doReturn(value).when(spy).method()` whenever stubbing `@Spy` partial mocks, or when overriding previously stubbed behaviors.
- Use `doThrow()`, `doNothing()`, `doAnswer()` for methods returning `void`.

---

### Testing Void Methods: State vs Behavior Verification

Methods returning `void` present a distinct unit testing challenge because they produce no return value that can be directly asserted using `assertEquals` or `assertThat`. Testing `void` methods requires identifying whether the method modifies internal/persistent state (**State Verification**) or delegates work to collaborating dependencies (**Behavior Verification**).

#### 1. State Verification (Observable State Mutation)

When a `void` method alters the internal state of the Class Under Test (CUT), the test should verify that state change via public query/getter methods or domain inspection.

```java
public class ShoppingCart {
    private final List<CartItem> items = new ArrayList<>();

    public void addItem(CartItem item) {
        if (item == null || item.price() <= 0) {
            throw new IllegalArgumentException("Invalid item");
        }
        this.items.add(item);
    }

    public List<CartItem> getItems() {
        return Collections.unmodifiableList(items);
    }
}

// State Verification Test
@Test
void addItem_ShouldIncrementCartSizeAndRetainItem() {
    ShoppingCart cart = new ShoppingCart();
    CartItem item = new CartItem("SKU-100", 25.00);

    cart.addItem(item);

    assertThat(cart.getItems())
        .hasSize(1)
        .containsExactly(item);
}
```

#### 2. Behavior Verification (Interaction Testing with Collaborators)

When a `void` method delegates tasks to external services or repositories (e.g., saving an entity to a repository, sending an email notification, or publishing a message to Kafka), the test must verify that the collaborator was invoked with the exact expected arguments.

##### Tools for Behavior Verification:
- `Mockito.verify(collaborator, times(n)).method(...)`: Verifies invocation count and matching arguments.
- `ArgumentCaptor<T>`: Captures the actual argument passed into the collaborator's method, allowing in-depth structural and property assertions on the payload.

```java
@ExtendWith(MockitoExtension.class)
class OrderProcessingServiceTest {

    @Mock
    private AuditLogRepository auditLogRepository;

    @Mock
    private NotificationClient notificationClient;

    @InjectMocks
    private OrderProcessingService orderProcessingService;

    @Captor
    private ArgumentCaptor<AuditEntry> auditEntryCaptor;

    @Test
    void completeOrder_ShouldPublishNotificationAndPersistAuditLog() {
        Order order = new Order("ORD-987", "CUSTOMER-42", 150.00);

        // Execute void method
        orderProcessingService.completeOrder(order);

        // 1. Verify interaction occurrence
        verify(notificationClient, times(1)).sendOrderConfirmation("CUSTOMER-42", "ORD-987");

        // 2. Capture and assert internal arguments passed to void collaborator
        verify(auditLogRepository, times(1)).saveAudit(auditEntryCaptor.capture());
        
        AuditEntry capturedEntry = auditEntryCaptor.getValue();
        assertThat(capturedEntry.getOrderId()).isEqualTo("ORD-987");
        assertThat(capturedEntry.getAction()).isEqualTo(AuditAction.ORDER_COMPLETED);
        assertThat(capturedEntry.getTimestamp()).isNotNull();
    }
}
```

#### 3. Exception Verification on Void Methods

When a `void` method is expected to fail given invalid arguments or degraded conditions, verify that the expected exception is thrown:

```java
@Test
void addItem_NullItem_ThrowsIllegalArgumentException() {
    ShoppingCart cart = new ShoppingCart();
    
    assertThrows(IllegalArgumentException.class, () -> cart.addItem(null));
}
```

#### Comparison: State Verification vs Behavior Verification

| Aspect | State Verification | Behavior Verification |
| :--- | :--- | :--- |
| **Primary Focus** | The result/state of the system after execution. | How the system interacted with its dependencies during execution. |
| **Coupling** | **Low coupling.** Resistant to internal implementation refactorings as long as public contract remains unchanged. | **Higher coupling.** Fragile if internal method call ordering or collaborator interactions change. |
| **Primary Tools** | AssertJ (`assertThat`), JUnit 5 (`assertEquals`, `assertTrue`). | Mockito (`verify()`, `verifyNoInteractions()`, `ArgumentCaptor`). |
| **Best Used For** | Domain entities, value objects, data structures, pure algorithmic logic. | Service layer orchestrators, event listeners, messaging dispatchers. |

---
