---
date: 2026-03-19 11:15:18
layout: post
title: "Decoupling the Core: An Introduction to Hexagonal Architecture"
subtitle: An introduction to ports and adapters, DDD, and the boundaries that
  keep Java backends maintainable
description: A practical guide to designing Spring Boot applications with
  Hexagonal Architecture, from domain modeling and use cases to adapters,
  testing, and long-term maintainability.
image: /assets/img/uploads/chatgpt-image-mar-19-2026-11_28_40-am.png
optimized_image: /assets/img/uploads/chatgpt-image-mar-19-2026-11_28_40-am.png
author: malkomich
permalink: /2026-03-19-decoupling-the-core-an-introduction-to-hexagonal-architecture/
category: software-arquitecture
tags:
  - architecture
  - cleancode
  - java
  - softwareengineering
  - backend
  - hexagonal
  - solid
paginate: false
---
## When Layered Architecture Stops Scaling

Layered architecture usually looks fine in the first month. The trouble starts when the service stops being a CRUD API and begins coordinating external providers, scheduled jobs, cache layers, persistence rules, and business constraints that are no longer trivial. At that point, the classic controller-service-repository stack tends to accumulate responsibilities in the wrong places.

That is where Hexagonal Architecture starts to make sense. Not because the diagram is elegant, but because it gives the design a rule that is easy to reason about: business logic stays in the center, infrastructure stays at the edges, and dependencies point inward.

## Why classic layering breaks down

Traditional layered systems are not wrong. They are just easier to get wrong as the application grows.

The usual failure mode is predictable:

* controllers start orchestrating use cases
* services become transaction scripts with framework knowledge
* repositories stop being persistence abstractions and begin to carry business decisions
* domain objects degrade into DTOs

The real problem is not the number of layers. The real problem is dependency direction. Once domain logic depends on JPA entities, Spring annotations, HTTP DTOs, or provider payloads, changing any outer concern becomes more expensive than it should be.

That cost shows up in very practical ways:

* tests become slow because business rules need full application context
* switching storage technology becomes invasive
* adding a second delivery mechanism, such as messaging or scheduling, duplicates orchestration logic
* external integration concerns leak into the core model

Hexagonal Architecture is a response to that coupling problem.

## The central idea of Hexagonal Architecture

![A hexagonal diagram showing the application core in the center with labeled ports (inbound and outbound) around the edges, connected to external adapters (REST Controller, Database, Message Queue, External API, etc.) on the outside. Should clearly show dependency arrows pointing inward toward the core.](https://miro.medium.com/1*aD3zDFzcF5Y2_27dvU213Q.png)

The pattern is simpler than the terminology sometimes suggests.

A hexagonal application has:

* a core with domain and use cases
* inbound ports, which define what the application can do
* outbound ports, which define what the application needs
* adapters, which connect those ports to HTTP, databases, queues, schedulers, external APIs, and any other technical detail

The point is not to create more interfaces for the sake of it. The point is to isolate the places where change is likely and costly.

Here is the idea in one sentence: **the application core should not know how data arrives, how it is stored, or which library is doing the work outside the boundary.**

### Inbound and outbound ports

Inbound ports model use cases:

```java
public interface SearchSlotsUseCase {
    List<PerformanceSlot> searchPublishedSlots(OffsetDateTime opensAfter, OffsetDateTime closesBefore);
}

public interface RefreshSlotCatalogUseCase {
    void refreshCatalog();
}
```

Outbound ports model dependencies:

```java
public interface SlotQueryRepository {
    List<PerformanceSlot> findPublishedSlots(SlotSearchCriteria criteria);
}

public interface SlotCommandRepository {
    void mergeSnapshot(List<PerformanceSlot> slots, Instant importedAt);
    void disableMissingSlots(Instant importedAt);
}

public interface PartnerCatalogClient {
    List<PerformanceSlot> fetchSlots();
}
```

Those interfaces are already saying something important. The application depends on business capabilities, not on `JpaRepository`, `RestTemplate`, Redis commands, or XML parsers.

## Hexagonal Architecture and DDD fit naturally together

Hexagonal Architecture and Domain-Driven Design solve different problems, but they work well together.

Hexagonal Architecture answers: **where should dependencies point?**

DDD answers: **how should the domain be modeled?**

In practice, the combination is useful because a clean architecture without a meaningful domain model quickly becomes ceremony, and a rich domain model without clear boundaries tends to get contaminated by infrastructure concerns.

### What DDD looks like in a service like this

You do not need the full DDD catalogue to get value. A few tactical choices are enough:

* name domain concepts in business language
* model small value objects when identity or meaning matters
* keep invariants close to the model
* represent domain errors explicitly

For example, if a slot coming from a partner catalog is identified by two related ids, that pair deserves its own type:

```java
public record PartnerSlotId(String seasonCode, Long slotNumber) {

    public PartnerSlotId {
        if (seasonCode == null || seasonCode.isBlank() || slotNumber == null) {
            throw new DomainValidationException("Partner slot id is required");
        }
    }
}
```

That is a better model than passing two unrelated `Long` values through every layer and hoping nobody swaps their order.

The same applies to the aggregate root or main entity:

```java
public record PerformanceSlot(
        UUID id,
        PartnerSlotId partnerSlotId,
        String headline,
        LocalDateTime opensAt,
        LocalDateTime closesAt,
        LocalDateTime bookingStartsAt,
        LocalDateTime bookingEndsAt,
        SalesChannel salesChannel,
        boolean soldOut,
        List<SeatBand> seatBands
) {

    public PerformanceSlot {
        if (headline == null || headline.isBlank()) {
            throw new DomainValidationException("headline is required");
        }
        if (opensAt == null || closesAt == null) {
            throw new DomainValidationException("slot dates are required");
        }
        if (opensAt.isAfter(closesAt)) {
            throw new InvalidDateRangeException("opensAt cannot be after closesAt");
        }
        if (bookingStartsAt != null && bookingEndsAt != null && bookingStartsAt.isAfter(bookingEndsAt)) {
            throw new InvalidDateRangeException("bookingStartsAt cannot be after bookingEndsAt");
        }
        seatBands = seatBands == null ? List.of() : List.copyOf(seatBands);
    }
}
```

This is where DDD pulls its weight. The model is not just data storage. It protects meaning and rejects invalid states.

### A pragmatic warning about DDD

This is also where many teams overdo it.

Not every object needs to become a rich domain object. Not every service needs domain events, factories, specifications, and anti-corruption layers from day one. If the domain is modest, forcing every DDD pattern into the design usually makes the code harder to follow.

The useful test is simple: does this abstraction make the business model clearer and safer, or is it only there because the pattern exists?

## A Spring Boot structure that stays readable

![A detailed component interaction diagram showing: Payment (domain entity) in the center, ProcessPaymentUseCase interface above it, SavePaymentPort interface below it, PaymentService (implementing ProcessPaymentUseCase) in the application layer, PaymentController (inbound adapter) connecting via use case, and PaymentPersistenceAdapter (outbound adapter) implementing the SavePaymentPort with PaymentJpaEntity. Use color coding and arrows to show interface implementations and dependencies.](https://miro.medium.com/1*ev1oXZACwF_up5fnDCvNhg.png)

A practical structure for a hexagonal Spring Boot service usually looks like this:

```text
src/main/java/com/example/catalog
├── domain
│   ├── exception
│   ├── model
│   └── port
├── application
│   ├── port/in
│   └── service
└── infrastructure
    ├── config
    ├── inbound
    └── outbound
```

The package names matter less than the rule behind them. Domain and application should not depend on infrastructure. Infrastructure is allowed to depend on the core.

I also think it is worth being slightly strict here: if your domain package imports JPA annotations or Spring MVC types, the architecture is already drifting.

## The application layer should orchestrate, not absorb infrastructure

The read side can stay very small:

```java
@Service
@RequiredArgsConstructor
public class SearchSlotsService implements SearchSlotsUseCase {

    private final SlotQueryRepository slotQueryRepository;

    @Override
    @Transactional(readOnly = true)
    public List<PerformanceSlot> searchPublishedSlots(OffsetDateTime opensAfter, OffsetDateTime closesBefore) {
        SlotSearchCriteria criteria = SlotSearchCriteria.of(
                opensAfter == null ? null : opensAfter.toLocalDateTime(),
                closesBefore == null ? null : closesBefore.toLocalDateTime()
        );

        return slotQueryRepository.findPublishedSlots(criteria);
    }
}
```

This is the right level of responsibility for an application service. It coordinates the use case, translates raw input into a domain concept, and delegates the actual query to an outbound port.

The write side usually shows the architecture more clearly:

```java
@Service
@RequiredArgsConstructor
public class RefreshSlotCatalogService implements RefreshSlotCatalogUseCase {

    private final PartnerCatalogClient partnerCatalogClient;
    private final SlotCommandRepository slotCommandRepository;
    private final Clock clock;

    @Override
    public void refreshCatalog() {
        Instant importedAt = Instant.now(clock);
        List<PerformanceSlot> slots = partnerCatalogClient.fetchSlots();

        slotCommandRepository.mergeSnapshot(slots, importedAt);
        slotCommandRepository.disableMissingSlots(importedAt);
    }
}
```

There are three details here that are easy to miss and worth keeping:

First, `Clock` is injected directly. I prefer that over inventing a `ClockPort` unless time itself is a business capability. Not every technical dependency deserves its own architectural boundary.

Second, the write port exposes business operations, not generic CRUD verbs. `mergeSnapshot` and `disableMissingSlots` describe a synchronization process much better than `save()` ever could.

Third, the service has no idea whether the provider uses XML, JSON, HTTP, gRPC, or a message queue. That uncertainty stays outside.

## Adapters should translate, not think

This is one of the most useful practical rules in a hexagonal system.

Adapters are translators. They convert transport models into domain objects and domain objects into transport or persistence models. They should be as boring as possible.

A persistence adapter can map between the domain model and JPA entities:

```java
@Component
@RequiredArgsConstructor
public class JpaSlotCommandRepositoryAdapter implements SlotCommandRepository {

    private final SpringDataSlotRepository repository;
    private final SlotEntityMapper mapper;

    @Override
    public void mergeSnapshot(List<PerformanceSlot> slots, Instant importedAt) {
        List<SlotEntity> entities = slots.stream()
                .map(slot -> mapper.toEntity(slot, importedAt))
                .toList();

        repository.saveAll(entities);
    }

    @Override
    public void disableMissingSlots(Instant importedAt) {
        repository.disableMissingSlots(importedAt);
    }
}
```

A provider adapter can isolate HTTP and parsing concerns:

```java
@Component
@RequiredArgsConstructor
public class PartnerCatalogClientAdapter implements PartnerCatalogClient {

    private final PartnerFeedGateway partnerFeedGateway;
    private final PartnerSlotFeedParser partnerSlotFeedParser;

    @Override
    public List<PerformanceSlot> fetchSlots() {
        String payload = partnerFeedGateway.fetchPayload();
        return partnerSlotFeedParser.parse(payload);
    }
}
```

This split matters because retry policy, HTTP timeouts, payload validation, caching, and fetch strategy are not domain concerns. They change for different reasons and should live in different adapters or configuration classes.

## CQRS often appears naturally here

A lot of articles treat CQRS like a dramatic architectural leap. In many Java backends, it appears in a much smaller and more useful form.

![Read and write flows in a hexagonal backend](/assets/img/uploads/hexagonal-read-write-flow.svg)

If reads and writes already have different behavior, different performance requirements, and different models, it is often enough to separate query and command ports.

That is already a CQRS-style decision:

```java
public interface SlotQueryRepository {
    List<PerformanceSlot> findPublishedSlots(SlotSearchCriteria criteria);
}

public interface SlotCommandRepository {
    void mergeSnapshot(List<PerformanceSlot> slots, Instant importedAt);
    void disableMissingSlots(Instant importedAt);
}
```

This is not full CQRS. There is no separate read database, no asynchronous projection system, and no event-driven choreography. But the separation is still valuable because it acknowledges that reads and writes have different semantics.

In practice, this buys you a lot:

* the read side can optimize for filtering, sorting, pagination, and caching
* the write side can optimize for consistency, idempotency, and synchronization logic
* tests become more focused because the contracts are narrower

This is one of those places where I think pragmatism matters more than purity. You do not need the full CQRS stack to benefit from the idea.

## SOLID and related patterns still matter

Hexagonal Architecture does not replace SOLID. It gives some of those principles a clearer architectural shape.

The most visible one is Dependency Inversion. Application services depend on abstractions such as `PartnerCatalogClient` or `SlotQueryRepository`, not on concrete adapters.

Single Responsibility also becomes easier to enforce when boundaries are explicit:

* controllers deal with HTTP
* schedulers trigger use cases
* adapters handle translation
* application services orchestrate
* domain objects protect business rules

Clean Architecture and Onion Architecture belong to the same family of ideas. The naming differs, but the core intention is similar: keep business rules in the center and push technology outward.

The useful takeaway is not which label you choose. The useful takeaway is whether your code actually follows the dependency rule you claim to use.

## Testing is where Hexagonal Architecture usually proves itself

The strongest argument for the pattern is rarely the diagram. It is the test suite.

A service designed around ports is much easier to test in layers:

* domain tests for invariants
* application service tests with mocked ports
* adapter integration tests for persistence or external clients
* controller tests for request and response mapping
* architecture tests to enforce dependency rules

A unit test for the synchronization flow can stay fast and precise:

```java
@ExtendWith(MockitoExtension.class)
class RefreshSlotCatalogServiceTest {

    @Mock
    private PartnerCatalogClient partnerCatalogClient;

    @Mock
    private SlotCommandRepository slotCommandRepository;

    @Mock
    private Clock clock;

    @InjectMocks
    private RefreshSlotCatalogService service;

    @Test
    void refreshUsesTheSameTimestampForMergeAndDisable() {
        Instant importedAt = Instant.parse("2026-01-01T00:00:00Z");
        when(clock.instant()).thenReturn(importedAt);
        when(partnerCatalogClient.fetchSlots()).thenReturn(List.of());

        service.refreshCatalog();

        verify(slotCommandRepository).mergeSnapshot(List.of(), importedAt);
        verify(slotCommandRepository).disableMissingSlots(importedAt);
    }
}
```

That test is quick, deterministic, and tied to business behavior instead of framework wiring.

I also strongly recommend architecture tests once the project has a few contributors. If the team says the domain must not depend on infrastructure, that rule should be executable:

```java
@ArchTest
static final ArchRule domainMustNotDependOnInfrastructure = noClasses()
        .that().resideInAPackage("..domain..")
        .should().dependOnClassesThat()
        .resideInAnyPackage("..application..", "..infrastructure..");
```

Without those checks, many projects keep the folder names and lose the architecture.

## Common mistakes that are worth avoiding early

There are a few mistakes I keep seeing in Java teams that adopt hexagonal structure too literally.

![Matrix architect talking about mistakes](/assets/img/uploads/architect.png)

### 1. Ports everywhere

If a dependency is not a real boundary, do not force it into a port. A `LoggerPort` is almost always a smell. A `ClockPort` often is too. Architectural boundaries should represent things that may vary independently or that materially affect testability and coupling.

### 2. Anemic domain with fancy packaging

A `domain` package full of records or Lombok DTOs with no invariants is not a domain model. It is just data moved to a different folder.

### 3. Business logic hidden in adapters

As soon as adapters start deciding what is valid, what should be retried, or how core workflows should behave, the architecture is being bypassed.

### 4. Pretending folder names equal architecture

Folders help, but they do not enforce anything by themselves. Dependency rules, code review discipline, and a small amount of architectural testing do.

## Conclusions

Hexagonal Architecture is not a silver bullet, and it is not the right answer for every application. For a small CRUD service with minimal business logic and few external dependencies, it might add more complexity than value. But for growing backends, especially in fast-moving industries like fintech, healthtech, or any domain with complex rules and frequent pivots, it is often the most solid long-term investment.

I've worked on systems where we've replaced entire infrastructure stacks, migrating from monoliths to microservices, switching message queues, or integrating new database systems, with minimal changes to core logic. That stability in the center while everything around it evolves is what makes the pattern powerful.

My advice: Start small and don't over-engineer. Apply the pattern to domains where volatility and growth risk are highest. If you're building a payment processing engine or a complex workflow system, hexagonal architecture makes sense from day one. If you're building a simple backend for a mobile app with CRUD operations, maybe start traditionally and refactor toward hexagonal boundaries as complexity grows. You can introduce ports and adapters incrementally, they're just interfaces and implementations, after all.