---
name: springboot-patterns
description: Spring Boot building blocks — layered architecture, REST controllers, JPA entities and repositories, services, and error handling
---

# Spring Boot patterns

## A layered architecture

```
src/main/java/com/example/app/
  config/          # @Configuration beans
  controller/      # @RestController (thin, delegates to service)
  service/         # @Service (business logic)
  repository/      # @Repository (data access via JPA)
  model/
    entity/        # @Entity JPA classes
    dto/           # Request/response DTOs
    mapper/        # MapStruct or manual mapping
  exception/       # @ControllerAdvice, custom exceptions
  security/        # SecurityFilterChain, JWT filters
```

Controllers own the HTTP layer, services hold the business logic, and repositories deal with persistence.

## A REST controller

```java
@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @GetMapping
    public Page<OrderResponse> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        return orderService.findAll(PageRequest.of(page, size));
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse create(@Valid @RequestBody CreateOrderRequest request) {
        return orderService.create(request);
    }

    @GetMapping("/{id}")
    public OrderResponse getById(@PathVariable UUID id) {
        return orderService.findById(id);
    }
}
```

## JPA entity and repository

```java
@Entity
@Table(name = "orders")
@Getter @Setter @NoArgsConstructor
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    @CreationTimestamp
    private Instant createdAt;
}

public interface OrderRepository extends JpaRepository<Order, UUID> {
    @Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.customer.id = :customerId")
    List<Order> findByCustomerWithItems(@Param("customerId") UUID customerId);

    @EntityGraph(attributePaths = {"customer", "items"})
    Optional<Order> findWithDetailsById(UUID id);
}
```

## The service layer

```java
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final OrderMapper orderMapper;
    private final EventPublisher eventPublisher;

    public OrderResponse findById(UUID id) {
        Order order = orderRepository.findWithDetailsById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Order", id));
        return orderMapper.toResponse(order);
    }

    @Transactional
    public OrderResponse create(CreateOrderRequest request) {
        Order order = orderMapper.toEntity(request);
        order = orderRepository.save(order);
        eventPublisher.publish(new OrderCreatedEvent(order.getId()));
        return orderMapper.toResponse(order);
    }
}
```

## A global exception handler

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail detail = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        Map<String, String> errors = ex.getFieldErrors().stream()
                .collect(Collectors.toMap(FieldError::getField, FieldError::getDefaultMessage));
        detail.setProperty("errors", errors);
        return detail;
    }
}
```

## What to avoid

- Wiring repositories straight into controllers and skipping the service layer
- Defaulting entity relationships to `FetchType.EAGER`
- Returning JPA entities from controllers instead of DTOs
- Read-only service methods that forget `@Transactional(readOnly = true)`
- Catching a blanket `Exception` instead of the specific type
- Hardcoding config values rather than using `@Value` or `@ConfigurationProperties`

## Before you ship

- [ ] Controllers stay thin and delegate to services
- [ ] JPA relationships default to `FetchType.LAZY`
- [ ] DTOs carry request/response, never raw entities
- [ ] `@Transactional` sits at the service layer with the right read/write scope
- [ ] Request DTOs carry validation annotations (`@Valid`, `@NotNull`, `@Size`)
- [ ] The global handler returns `ProblemDetail` (RFC 7807)
- [ ] Entity graphs or `JOIN FETCH` head off N+1 queries
- [ ] Integration tests run under `@SpringBootTest` with test containers
