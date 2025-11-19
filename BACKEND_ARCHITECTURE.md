# FitHub Backend Architecture
## Kotlin Spring Boot 3.5.6 with JDK 21

**Version:** 1.0
**Last Updated:** November 2025
**Authors:** FitHub Engineering Team

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Technology Stack](#technology-stack)
3. [Architectural Patterns](#architectural-patterns)
4. [Clean/Hexagonal Architecture](#cleanhexagonal-architecture)
5. [Domain-Driven Design (DDD)](#domain-driven-design-ddd)
6. [Multi-Layered Structure](#multi-layered-structure)
7. [Event-Driven Architecture](#event-driven-architecture)
8. [CQRS Pattern](#cqrs-pattern)
9. [Project Structure](#project-structure)
10. [Data Architecture](#data-architecture)
11. [Security Architecture](#security-architecture)
12. [Deployment Architecture](#deployment-architecture)
13. [Architectural Decision Records](#architectural-decision-records)

---

## Executive Summary

FitHub's backend architecture is designed to support a **multi-tenant SaaS platform** for fitness facility management in Saudi Arabia. The system must handle:

- **Multi-tenancy:** Isolated data and operations for multiple fitness facilities
- **Cultural compliance:** Arabic language support, Hijri calendar, prayer time integration
- **Regulatory compliance:** ZATCA e-invoicing, PDPL data protection, Saudization tracking
- **Scalability:** Growing from 10 to 1000+ facilities
- **High availability:** 99.9% uptime SLA
- **Complex business logic:** Family memberships, gender-segregated scheduling, tiered pricing

### Core Architectural Principles

1. **Separation of Concerns:** Clear boundaries between business logic and infrastructure
2. **Dependency Inversion:** Domain layer depends on nothing; all dependencies point inward
3. **Testability:** Every component can be tested in isolation
4. **Maintainability:** Code organized by business capability, not technical layer
5. **Scalability:** Event-driven architecture enables horizontal scaling
6. **Flexibility:** Hexagonal architecture allows swapping implementations without changing business logic

---

## Technology Stack

### Core Framework
- **Kotlin 2.0+**: Modern, concise, null-safe JVM language
- **JDK 21**: Latest LTS with virtual threads, pattern matching, and performance improvements
- **Spring Boot 3.5.6**: Enterprise-grade framework with modern Jakarta EE support

### Spring Ecosystem
```kotlin
dependencies {
    // Core Spring Boot
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.springframework.boot:spring-boot-starter-actuator")

    // Event-Driven Architecture
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("org.springframework.kafka:spring-kafka")

    // Database
    implementation("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    // Observability
    implementation("io.micrometer:micrometer-registry-prometheus")
    implementation("io.micrometer:micrometer-tracing-bridge-brave")

    // Documentation
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")

    // Kotlin Support
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")

    // Testing
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("io.mockk:mockk:1.13.8")
    testImplementation("io.kotest:kotest-runner-junit5:5.8.0")
    testImplementation("org.testcontainers:postgresql:1.19.3")
}
```

### Rationale for Technology Choices

#### Why Kotlin over Java?
1. **Null Safety:** Eliminates NullPointerExceptions at compile time
2. **Conciseness:** 40% less boilerplate than Java
3. **Coroutines:** Native async/await support for reactive programming
4. **Data Classes:** Built-in immutable value objects perfect for DDD
5. **Extension Functions:** Add behavior without inheritance
6. **Smart Casts:** Type-safe pattern matching
7. **Multiplatform:** Share code with mobile applications

#### Why JDK 21?
1. **Virtual Threads (Project Loom):** Massive concurrency improvements for blocking I/O
2. **Pattern Matching:** Cleaner, more maintainable code
3. **Record Patterns:** Enhanced sealed classes support
4. **Performance:** G1GC improvements, faster startup
5. **Long-term Support:** Stable until 2028

#### Why Spring Boot 3.5.6?
1. **Jakarta EE 10:** Modern enterprise standards
2. **Native GraalVM Support:** Fast startup, low memory footprint
3. **Observability:** Built-in metrics, tracing, and logging
4. **AOT Compilation:** Ahead-of-time optimization
5. **Virtual Threads Support:** Seamless integration with JDK 21

---

## Architectural Patterns

### 1. Clean/Hexagonal Architecture

FitHub implements **Hexagonal Architecture (Ports and Adapters)** combined with **Clean Architecture** principles to achieve:

- **Technology Independence:** Business logic unaware of frameworks, databases, or UI
- **Testability:** Core domain can be tested without external dependencies
- **Flexibility:** Easy to swap databases, message brokers, or web frameworks
- **Maintainability:** Changes in infrastructure don't affect business rules

#### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                        │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌───────────┐ │
│  │    REST    │  │   GraphQL  │  │    gRPC    │  │  WebSocket│ │
│  │ Controllers│  │   Resolvers│  │   Services │  │  Handlers │ │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘  └─────┬─────┘ │
└────────┼───────────────┼───────────────┼───────────────┼────────┘
         │               │               │               │
         └───────────────┴───────────────┴───────────────┘
                                 │
         ┌───────────────────────┴───────────────────────┐
         │          APPLICATION LAYER (Use Cases)         │
         │  ┌──────────────────────────────────────────┐ │
         │  │        Input Port (Interface)            │ │
         │  │  - CreateMembershipUseCase               │ │
         │  │  - ProcessPaymentUseCase                 │ │
         │  │  - ScheduleClassUseCase                  │ │
         │  └──────────────┬───────────────────────────┘ │
         │                 │                              │
         │  ┌──────────────▼───────────────────────────┐ │
         │  │        Use Case Implementations          │ │
         │  │  - CreateMembershipService               │ │
         │  │  - ProcessPaymentService                 │ │
         │  │  - ScheduleClassService                  │ │
         │  └──────────────┬───────────────────────────┘ │
         └─────────────────┼───────────────────────────┬─┘
                           │                           │
         ┌─────────────────▼───────────────────────────▼─┐
         │              DOMAIN LAYER (Core)              │
         │  ┌──────────────────────────────────────────┐ │
         │  │             Entities                     │ │
         │  │  - Member, Membership, Facility          │ │
         │  │  - Class, Equipment, Payment             │ │
         │  └──────────────────────────────────────────┘ │
         │  ┌──────────────────────────────────────────┐ │
         │  │         Value Objects                    │ │
         │  │  - MemberId, Email, Money, Period        │ │
         │  └──────────────────────────────────────────┘ │
         │  ┌──────────────────────────────────────────┐ │
         │  │        Domain Services                   │ │
         │  │  - PricingService, SchedulingService     │ │
         │  └──────────────────────────────────────────┘ │
         │  ┌──────────────────────────────────────────┐ │
         │  │        Domain Events                     │ │
         │  │  - MembershipCreated, PaymentProcessed   │ │
         │  └──────────────────────────────────────────┘ │
         │  ┌──────────────────────────────────────────┐ │
         │  │     Output Ports (Interfaces)            │ │
         │  │  - MemberRepository                      │ │
         │  │  - PaymentGateway                        │ │
         │  │  - NotificationService                   │ │
         │  └──────────────┬───────────────────────────┘ │
         └─────────────────┼──────────────────────────────┘
                           │
         ┌─────────────────▼───────────────────────────┐
         │         INFRASTRUCTURE LAYER (Adapters)     │
         │  ┌──────────────────────────────────────────┐ │
         │  │        Output Adapters                   │ │
         │  │  - JpaMemberRepository                   │ │
         │  │  - StripePaymentGatewayAdapter           │ │
         │  │  - TwilioNotificationAdapter             │ │
         │  │  - KafkaEventPublisher                   │ │
         │  └──────────────────────────────────────────┘ │
         │  ┌──────────────────────────────────────────┐ │
         │  │           Persistence                    │ │
         │  │  - PostgreSQL, Redis, Kafka              │ │
         │  └──────────────────────────────────────────┘ │
         │  ┌──────────────────────────────────────────┐ │
         │  │        External Services                 │ │
         │  │  - ZATCA, Payment Providers, SMS         │ │
         │  └──────────────────────────────────────────┘ │
         └──────────────────────────────────────────────┘
```

#### Dependency Rule

**All dependencies point inward.** The domain layer has zero dependencies on outer layers.

```kotlin
// ✅ CORRECT: Application layer depends on Domain
class CreateMembershipService(
    private val memberRepository: MemberRepository // Domain interface
) : CreateMembershipUseCase { // Domain interface
    override fun execute(command: CreateMembershipCommand): Membership {
        // Business logic using domain objects
    }
}

// ❌ INCORRECT: Domain depending on Infrastructure
class Member(
    @Entity // JPA annotation - infrastructure concern!
    @Table(name = "members")
    val id: MemberId
)
```

---

## Domain-Driven Design (DDD)

### Strategic Design

#### Bounded Contexts

FitHub is divided into **7 bounded contexts**, each with its own domain model:

```
┌──────────────────────────────────────────────────────────────────┐
│                        FITHUB SYSTEM                             │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │   MEMBERSHIP   │  │    FACILITY    │  │     STAFF      │    │
│  │    CONTEXT     │  │    CONTEXT     │  │    CONTEXT     │    │
│  │                │  │                │  │                │    │
│  │ - Member       │  │ - Facility     │  │ - Employee     │    │
│  │ - Membership   │  │ - Location     │  │ - Schedule     │    │
│  │ - Family       │  │ - Section      │  │ - Role         │    │
│  │ - Enrollment   │  │ - Amenity      │  │ - Permission   │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
│                                                                  │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐    │
│  │     CLASS      │  │   EQUIPMENT    │  │    PAYMENT     │    │
│  │    CONTEXT     │  │    CONTEXT     │  │    CONTEXT     │    │
│  │                │  │                │  │                │    │
│  │ - Class        │  │ - Equipment    │  │ - Payment      │    │
│  │ - Session      │  │ - Category     │  │ - Invoice      │    │
│  │ - Schedule     │  │ - Maintenance  │  │ - Transaction  │    │
│  │ - Attendance   │  │ - Status       │  │ - Subscription │    │
│  └────────────────┘  └────────────────┘  └────────────────┘    │
│                                                                  │
│  ┌────────────────┐                                             │
│  │   NOTIFICATION │                                             │
│  │    CONTEXT     │                                             │
│  │                │                                             │
│  │ - Notification │                                             │
│  │ - Template     │                                             │
│  │ - Channel      │                                             │
│  │ - Preference   │                                             │
│  └────────────────┘                                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

**Context Mapping:**

```kotlin
// Anti-Corruption Layer between contexts
class MembershipPaymentMapper {
    fun toPaymentContext(member: membership.Member): payment.Customer {
        return payment.Customer(
            customerId = CustomerId(member.id.value),
            fullName = member.fullName,
            email = member.email
        )
    }
}
```

#### Ubiquitous Language

All code uses the **exact same terminology** as domain experts:

| Ubiquitous Term | Implementation | Avoid |
|-----------------|----------------|-------|
| Member | `Member` | User, Customer, Client |
| Membership | `Membership` | Subscription, Plan |
| Family Account | `FamilyAccount` | Group, Bundle |
| Prayer Time | `PrayerTime` | Break, Pause |
| Hijri Date | `HijriDate` | Islamic Date, Lunar Date |
| Saudization Rate | `SaudizationRate` | Localization Percentage |
| Gender-Segregated | `GenderSegregation` | Separate, Divided |

### Tactical Design

#### 1. Entities

**Entities have identity** that persists across state changes:

```kotlin
/**
 * Member entity - represents a gym member with a unique identity.
 * The member's identity (MemberId) remains constant even when attributes change.
 */
data class Member(
    val id: MemberId,
    val facilityId: FacilityId,
    val personalInfo: PersonalInfo,
    val contactInfo: ContactInfo,
    val membershipStatus: MembershipStatus,
    val enrolledClasses: List<ClassEnrollment>,
    val familyAccountId: FamilyAccountId? = null,
    val createdAt: Instant,
    val updatedAt: Instant
) {
    companion object {
        fun create(
            facilityId: FacilityId,
            personalInfo: PersonalInfo,
            contactInfo: ContactInfo
        ): Member {
            return Member(
                id = MemberId.generate(),
                facilityId = facilityId,
                personalInfo = personalInfo,
                contactInfo = contactInfo,
                membershipStatus = MembershipStatus.PENDING,
                enrolledClasses = emptyList(),
                familyAccountId = null,
                createdAt = Instant.now(),
                updatedAt = Instant.now()
            )
        }
    }

    fun activate(membership: Membership): Member {
        require(membershipStatus == MembershipStatus.PENDING) {
            "Cannot activate member with status $membershipStatus"
        }
        return copy(
            membershipStatus = MembershipStatus.ACTIVE,
            updatedAt = Instant.now()
        )
    }

    fun enrollInClass(classSession: ClassSession): Member {
        require(canEnroll(classSession)) {
            "Member cannot enroll in class: ${classSession.id}"
        }

        val enrollment = ClassEnrollment.create(
            memberId = id,
            classSessionId = classSession.id,
            enrolledAt = Instant.now()
        )

        return copy(
            enrolledClasses = enrolledClasses + enrollment,
            updatedAt = Instant.now()
        )
    }

    private fun canEnroll(classSession: ClassSession): Boolean {
        return membershipStatus == MembershipStatus.ACTIVE &&
               !hasConflictingClass(classSession) &&
               classSession.hasAvailableCapacity()
    }

    private fun hasConflictingClass(classSession: ClassSession): Boolean {
        return enrolledClasses.any { it.conflictsWith(classSession) }
    }
}
```

#### 2. Value Objects

**Value Objects have no identity** - equality is based on attributes:

```kotlin
/**
 * Money value object - immutable representation of monetary values.
 * Two Money objects with the same amount and currency are equal.
 */
@JvmInline
value class Money(val amount: BigDecimal, val currency: Currency = Currency.SAR) {
    init {
        require(amount >= BigDecimal.ZERO) {
            "Money amount cannot be negative: $amount"
        }
    }

    operator fun plus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot add money with different currencies"
        }
        return Money(amount + other.amount, currency)
    }

    operator fun minus(other: Money): Money {
        require(currency == other.currency) {
            "Cannot subtract money with different currencies"
        }
        return Money(amount - other.amount, currency)
    }

    operator fun times(multiplier: Int): Money {
        return Money(amount * multiplier.toBigDecimal(), currency)
    }

    fun isGreaterThan(other: Money): Boolean {
        require(currency == other.currency)
        return amount > other.amount
    }
}

enum class Currency(val code: String, val symbol: String) {
    SAR("SAR", "﷼"),  // Saudi Riyal
    USD("USD", "$")
}

/**
 * Email value object with built-in validation.
 */
@JvmInline
value class Email(val value: String) {
    init {
        require(isValid(value)) {
            "Invalid email format: $value"
        }
    }

    companion object {
        private val EMAIL_REGEX = """^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$""".toRegex()

        fun isValid(email: String): Boolean = EMAIL_REGEX.matches(email)
    }
}

/**
 * HijriDate value object for Islamic calendar support.
 */
data class HijriDate(
    val year: Int,
    val month: HijriMonth,
    val day: Int
) {
    init {
        require(year > 0) { "Year must be positive" }
        require(day in 1..30) { "Day must be between 1 and 30" }
    }

    fun toGregorian(): LocalDate {
        // Implementation using Umm Al-Qura calendar
        return HijriCalendarConverter.toGregorian(this)
    }

    companion object {
        fun fromGregorian(gregorianDate: LocalDate): HijriDate {
            return HijriCalendarConverter.fromGregorian(gregorianDate)
        }

        fun now(): HijriDate = fromGregorian(LocalDate.now())
    }
}

enum class HijriMonth(val arabicName: String) {
    MUHARRAM("محرم"),
    SAFAR("صفر"),
    RABI_AL_AWWAL("ربيع الأول"),
    RABI_AL_THANI("ربيع الثاني"),
    JUMADA_AL_AWWAL("جمادى الأولى"),
    JUMADA_AL_THANI("جمادى الثانية"),
    RAJAB("رجب"),
    SHABAN("شعبان"),
    RAMADAN("رمضان"),
    SHAWWAL("شوال"),
    DHU_AL_QIDAH("ذو القعدة"),
    DHU_AL_HIJJAH("ذو الحجة")
}
```

#### 3. Aggregates

**Aggregates enforce consistency boundaries:**

```kotlin
/**
 * FamilyAccount aggregate root.
 * Ensures consistency for all family members and their shared membership.
 *
 * Business Rules:
 * - Primary member must exist
 * - Maximum 10 family members
 * - All members share the same facility
 * - One active family membership at a time
 */
data class FamilyAccount(
    val id: FamilyAccountId,
    val facilityId: FacilityId,
    val primaryMemberId: MemberId,
    val familyMembers: List<FamilyMember>,
    val familyMembership: FamilyMembership?,
    val createdAt: Instant,
    val updatedAt: Instant
) {
    companion object {
        const val MAX_FAMILY_MEMBERS = 10

        fun create(
            facilityId: FacilityId,
            primaryMemberId: MemberId
        ): FamilyAccount {
            return FamilyAccount(
                id = FamilyAccountId.generate(),
                facilityId = facilityId,
                primaryMemberId = primaryMemberId,
                familyMembers = listOf(
                    FamilyMember(
                        memberId = primaryMemberId,
                        relationship = FamilyRelationship.PRIMARY,
                        addedAt = Instant.now()
                    )
                ),
                familyMembership = null,
                createdAt = Instant.now(),
                updatedAt = Instant.now()
            )
        }
    }

    fun addFamilyMember(
        memberId: MemberId,
        relationship: FamilyRelationship
    ): FamilyAccount {
        require(familyMembers.size < MAX_FAMILY_MEMBERS) {
            "Family account cannot have more than $MAX_FAMILY_MEMBERS members"
        }

        require(familyMembers.none { it.memberId == memberId }) {
            "Member $memberId is already in the family account"
        }

        val newMember = FamilyMember(
            memberId = memberId,
            relationship = relationship,
            addedAt = Instant.now()
        )

        return copy(
            familyMembers = familyMembers + newMember,
            updatedAt = Instant.now()
        )
    }

    fun activateFamilyMembership(membership: FamilyMembership): FamilyAccount {
        require(familyMembership == null || !familyMembership.isActive()) {
            "Family account already has an active membership"
        }

        require(membership.familyAccountId == id) {
            "Membership does not belong to this family account"
        }

        return copy(
            familyMembership = membership,
            updatedAt = Instant.now()
        )
    }

    fun removeFamilyMember(memberId: MemberId): FamilyAccount {
        require(memberId != primaryMemberId) {
            "Cannot remove primary member from family account"
        }

        return copy(
            familyMembers = familyMembers.filter { it.memberId != memberId },
            updatedAt = Instant.now()
        )
    }
}

data class FamilyMember(
    val memberId: MemberId,
    val relationship: FamilyRelationship,
    val addedAt: Instant
)

enum class FamilyRelationship {
    PRIMARY,
    SPOUSE,
    CHILD,
    PARENT,
    SIBLING
}
```

#### 4. Domain Services

**Domain Services contain logic that doesn't naturally fit in any entity:**

```kotlin
/**
 * PricingService - calculates membership prices based on complex business rules.
 *
 * Pricing Rules:
 * - Family memberships get 20% discount
 * - Annual memberships get 2 months free
 * - Special Ramadan pricing (30% discount during Ramadan)
 * - Gender-segregated facilities may have different pricing
 */
class PricingService(
    private val hijriCalendar: HijriCalendarService
) {
    fun calculateMembershipPrice(
        membershipType: MembershipType,
        billingPeriod: BillingPeriod,
        isFamilyMembership: Boolean,
        facilityPricingTier: PricingTier
    ): Money {
        val basePrice = getBasePrice(membershipType, facilityPricingTier)
        var finalPrice = applyBillingPeriodDiscount(basePrice, billingPeriod)

        if (isFamilyMembership) {
            finalPrice = applyFamilyDiscount(finalPrice)
        }

        if (isRamadan()) {
            finalPrice = applyRamadanDiscount(finalPrice)
        }

        return finalPrice
    }

    private fun getBasePrice(
        membershipType: MembershipType,
        tier: PricingTier
    ): Money {
        return when (membershipType) {
            MembershipType.BASIC -> tier.basicPrice
            MembershipType.PREMIUM -> tier.premiumPrice
            MembershipType.VIP -> tier.vipPrice
        }
    }

    private fun applyBillingPeriodDiscount(
        price: Money,
        period: BillingPeriod
    ): Money {
        return when (period) {
            BillingPeriod.MONTHLY -> price
            BillingPeriod.QUARTERLY -> price * 3 * 0.95.toBigDecimal()
            BillingPeriod.ANNUALLY -> price * 10 // 2 months free
        }
    }

    private fun applyFamilyDiscount(price: Money): Money {
        return price * 0.80.toBigDecimal() // 20% discount
    }

    private fun applyRamadanDiscount(price: Money): Money {
        return price * 0.70.toBigDecimal() // 30% discount
    }

    private fun isRamadan(): Boolean {
        val today = hijriCalendar.today()
        return today.month == HijriMonth.RAMADAN
    }
}

/**
 * SchedulingService - handles complex scheduling logic with prayer times.
 */
class SchedulingService(
    private val prayerTimeProvider: PrayerTimeProvider
) {
    fun canScheduleClass(
        startTime: Instant,
        duration: Duration,
        location: Location
    ): SchedulingResult {
        val endTime = startTime.plus(duration)
        val prayerTimes = prayerTimeProvider.getPrayerTimes(
            date = LocalDate.ofInstant(startTime, ZoneId.systemDefault()),
            location = location
        )

        // Check if class conflicts with prayer times
        val conflictingPrayer = prayerTimes.find { prayer ->
            val prayerWindow = TimeWindow(
                start = prayer.time,
                end = prayer.time.plus(Duration.ofMinutes(30))
            )
            prayerWindow.overlapsWith(TimeWindow(startTime, endTime))
        }

        return if (conflictingPrayer != null) {
            SchedulingResult.PrayerTimeConflict(conflictingPrayer)
        } else {
            SchedulingResult.Success
        }
    }
}

sealed class SchedulingResult {
    object Success : SchedulingResult()
    data class PrayerTimeConflict(val prayer: PrayerTime) : SchedulingResult()
    data class FacilityUnavailable(val reason: String) : SchedulingResult()
}
```

#### 5. Domain Events

**Domain Events capture important business moments:**

```kotlin
/**
 * Base interface for all domain events.
 */
sealed interface DomainEvent {
    val eventId: EventId
    val aggregateId: String
    val occurredAt: Instant
    val eventType: String
}

/**
 * MembershipActivated - published when a membership becomes active.
 */
data class MembershipActivated(
    override val eventId: EventId = EventId.generate(),
    override val aggregateId: String,
    override val occurredAt: Instant = Instant.now(),
    val membershipId: MembershipId,
    val memberId: MemberId,
    val facilityId: FacilityId,
    val membershipType: MembershipType,
    val startDate: LocalDate,
    val endDate: LocalDate
) : DomainEvent {
    override val eventType: String = "MembershipActivated"
}

/**
 * PaymentProcessed - published when a payment is successfully processed.
 */
data class PaymentProcessed(
    override val eventId: EventId = EventId.generate(),
    override val aggregateId: String,
    override val occurredAt: Instant = Instant.now(),
    val paymentId: PaymentId,
    val memberId: MemberId,
    val amount: Money,
    val paymentMethod: PaymentMethod,
    val transactionId: String
) : DomainEvent {
    override val eventType: String = "PaymentProcessed"
}

/**
 * ClassSessionCompleted - published when a class session ends.
 */
data class ClassSessionCompleted(
    override val eventId: EventId = EventId.generate(),
    override val aggregateId: String,
    override val occurredAt: Instant = Instant.now(),
    val classSessionId: ClassSessionId,
    val classId: ClassId,
    val facilityId: FacilityId,
    val instructorId: EmployeeId,
    val attendanceCount: Int,
    val completedAt: Instant
) : DomainEvent {
    override val eventType: String = "ClassSessionCompleted"
}
```

#### 6. Repositories (Domain Interfaces)

**Repositories are defined in the domain but implemented in infrastructure:**

```kotlin
/**
 * MemberRepository - domain interface for member persistence.
 * Implementation is in infrastructure layer.
 */
interface MemberRepository {
    fun save(member: Member): Member
    fun findById(id: MemberId): Member?
    fun findByFacilityId(facilityId: FacilityId): List<Member>
    fun findByEmail(email: Email): Member?
    fun existsByEmail(email: Email): Boolean
    fun delete(id: MemberId)
}

/**
 * FamilyAccountRepository - handles family account persistence.
 */
interface FamilyAccountRepository {
    fun save(familyAccount: FamilyAccount): FamilyAccount
    fun findById(id: FamilyAccountId): FamilyAccount?
    fun findByPrimaryMemberId(memberId: MemberId): FamilyAccount?
    fun findByFacilityId(facilityId: FacilityId): List<FamilyAccount>
}

/**
 * ClassSessionRepository - manages class session data.
 */
interface ClassSessionRepository {
    fun save(classSession: ClassSession): ClassSession
    fun findById(id: ClassSessionId): ClassSession?
    fun findUpcomingSessions(
        facilityId: FacilityId,
        startDate: LocalDate,
        endDate: LocalDate
    ): List<ClassSession>
    fun findByInstructor(
        instructorId: EmployeeId,
        from: Instant,
        to: Instant
    ): List<ClassSession>
}
```

---

## Multi-Layered Structure

### Layer Responsibilities

```
┌──────────────────────────────────────────────────────────────┐
│                    PRESENTATION LAYER                         │
│  Responsibility: Handle HTTP requests/responses              │
│  Dependencies: Application Layer                             │
│  Components: Controllers, DTOs, Exception Handlers           │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                          │
│  Responsibility: Orchestrate use cases                       │
│  Dependencies: Domain Layer                                  │
│  Components: Use Cases, Application Services, Mappers        │
└──────────────────────────────────────────────────────────────┘
                              ↓
┌──────────────────────────────────────────────────────────────┐
│                     DOMAIN LAYER                             │
│  Responsibility: Business logic and rules                    │
│  Dependencies: NONE (completely independent)                 │
│  Components: Entities, Value Objects, Domain Services        │
└──────────────────────────────────────────────────────────────┘
                              ↑
┌──────────────────────────────────────────────────────────────┐
│                 INFRASTRUCTURE LAYER                         │
│  Responsibility: Technical capabilities                      │
│  Dependencies: Domain Layer (via interfaces)                 │
│  Components: Repositories, External Services, Configs        │
└──────────────────────────────────────────────────────────────┘
```

### 1. Presentation Layer

**Responsibility:** Translate HTTP requests into application commands and domain models into HTTP responses.

```kotlin
/**
 * REST Controller for member management endpoints.
 */
@RestController
@RequestMapping("/api/v1/members")
@Validated
class MemberController(
    private val createMemberUseCase: CreateMemberUseCase,
    private val getMemberUseCase: GetMemberUseCase,
    private val updateMemberUseCase: UpdateMemberUseCase
) {

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun createMember(
        @RequestHeader("X-Facility-Id") facilityId: String,
        @Valid @RequestBody request: CreateMemberRequest
    ): MemberResponse {
        val command = CreateMemberCommand(
            facilityId = FacilityId(facilityId),
            arabicName = request.arabicName,
            englishName = request.englishName,
            email = Email(request.email),
            phone = PhoneNumber(request.phone),
            nationalId = NationalId(request.nationalId),
            dateOfBirth = request.dateOfBirth,
            gender = request.gender
        )

        val member = createMemberUseCase.execute(command)
        return member.toResponse()
    }

    @GetMapping("/{memberId}")
    fun getMember(
        @RequestHeader("X-Facility-Id") facilityId: String,
        @PathVariable memberId: String
    ): MemberResponse {
        val query = GetMemberQuery(
            facilityId = FacilityId(facilityId),
            memberId = MemberId(memberId)
        )

        val member = getMemberUseCase.execute(query)
            ?: throw MemberNotFoundException(memberId)

        return member.toResponse()
    }

    @PutMapping("/{memberId}")
    fun updateMember(
        @RequestHeader("X-Facility-Id") facilityId: String,
        @PathVariable memberId: String,
        @Valid @RequestBody request: UpdateMemberRequest
    ): MemberResponse {
        val command = UpdateMemberCommand(
            facilityId = FacilityId(facilityId),
            memberId = MemberId(memberId),
            contactInfo = ContactInfo(
                email = Email(request.email),
                phone = PhoneNumber(request.phone),
                address = Address(
                    street = request.address.street,
                    city = request.address.city,
                    postalCode = request.address.postalCode
                )
            )
        )

        val member = updateMemberUseCase.execute(command)
        return member.toResponse()
    }
}

/**
 * Data Transfer Objects (DTOs) for API requests/responses.
 */
data class CreateMemberRequest(
    @field:NotBlank(message = "Arabic name is required")
    val arabicName: String,

    @field:NotBlank(message = "English name is required")
    val englishName: String,

    @field:Email(message = "Invalid email format")
    val email: String,

    @field:Pattern(
        regexp = "^\\+966[0-9]{9}$",
        message = "Phone must be Saudi number (+966XXXXXXXXX)"
    )
    val phone: String,

    @field:Pattern(
        regexp = "^[12][0-9]{9}$",
        message = "Invalid Saudi national ID"
    )
    val nationalId: String,

    @field:Past(message = "Date of birth must be in the past")
    val dateOfBirth: LocalDate,

    @field:NotNull(message = "Gender is required")
    val gender: Gender
)

data class MemberResponse(
    val id: String,
    val arabicName: String,
    val englishName: String,
    val email: String,
    val phone: String,
    val membershipStatus: String,
    val createdAt: Instant,
    val updatedAt: Instant
)

/**
 * Extension function to convert domain model to response DTO.
 */
fun Member.toResponse(): MemberResponse {
    return MemberResponse(
        id = id.value,
        arabicName = personalInfo.arabicName,
        englishName = personalInfo.englishName,
        email = contactInfo.email.value,
        phone = contactInfo.phone.value,
        membershipStatus = membershipStatus.name,
        createdAt = createdAt,
        updatedAt = updatedAt
    )
}

/**
 * Global exception handler for consistent error responses.
 */
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(MemberNotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleMemberNotFound(ex: MemberNotFoundException): ErrorResponse {
        return ErrorResponse(
            code = "MEMBER_NOT_FOUND",
            message = ex.message ?: "Member not found",
            timestamp = Instant.now()
        )
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidationErrors(ex: MethodArgumentNotValidException): ErrorResponse {
        val errors = ex.bindingResult.fieldErrors.map {
            FieldError(field = it.field, message = it.defaultMessage ?: "Invalid value")
        }

        return ErrorResponse(
            code = "VALIDATION_ERROR",
            message = "Request validation failed",
            fieldErrors = errors,
            timestamp = Instant.now()
        )
    }

    @ExceptionHandler(DomainException::class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    fun handleDomainException(ex: DomainException): ErrorResponse {
        return ErrorResponse(
            code = ex.errorCode,
            message = ex.message ?: "Business rule violation",
            timestamp = Instant.now()
        )
    }
}

data class ErrorResponse(
    val code: String,
    val message: String,
    val fieldErrors: List<FieldError> = emptyList(),
    val timestamp: Instant
)

data class FieldError(
    val field: String,
    val message: String
)
```

### 2. Application Layer

**Responsibility:** Implement use cases by coordinating domain objects and infrastructure services.

```kotlin
/**
 * Use case interface (port) defined in domain layer.
 */
interface CreateMemberUseCase {
    fun execute(command: CreateMemberCommand): Member
}

/**
 * Command object encapsulating input data.
 */
data class CreateMemberCommand(
    val facilityId: FacilityId,
    val arabicName: String,
    val englishName: String,
    val email: Email,
    val phone: PhoneNumber,
    val nationalId: NationalId,
    val dateOfBirth: LocalDate,
    val gender: Gender
)

/**
 * Use case implementation in application layer.
 */
@Service
@Transactional
class CreateMemberService(
    private val memberRepository: MemberRepository,
    private val facilityRepository: FacilityRepository,
    private val eventPublisher: DomainEventPublisher,
    private val notificationService: NotificationService
) : CreateMemberUseCase {

    private val logger = LoggerFactory.getLogger(javaClass)

    override fun execute(command: CreateMemberCommand): Member {
        logger.info("Creating member for facility: ${command.facilityId}")

        // 1. Validate facility exists
        val facility = facilityRepository.findById(command.facilityId)
            ?: throw FacilityNotFoundException(command.facilityId)

        // 2. Check for duplicate email
        if (memberRepository.existsByEmail(command.email)) {
            throw DuplicateEmailException(command.email)
        }

        // 3. Create domain entity
        val member = Member.create(
            facilityId = command.facilityId,
            personalInfo = PersonalInfo(
                arabicName = command.arabicName,
                englishName = command.englishName,
                nationalId = command.nationalId,
                dateOfBirth = command.dateOfBirth,
                gender = command.gender
            ),
            contactInfo = ContactInfo(
                email = command.email,
                phone = command.phone
            )
        )

        // 4. Persist to database
        val savedMember = memberRepository.save(member)

        // 5. Publish domain event
        val event = MemberCreated(
            aggregateId = savedMember.id.value,
            memberId = savedMember.id,
            facilityId = savedMember.facilityId,
            email = savedMember.contactInfo.email,
            occurredAt = Instant.now()
        )
        eventPublisher.publish(event)

        // 6. Send welcome notification (async)
        notificationService.sendWelcomeEmail(savedMember)

        logger.info("Member created successfully: ${savedMember.id}")

        return savedMember
    }
}

/**
 * Query use case for reading member data (CQRS read side).
 */
interface GetMemberUseCase {
    fun execute(query: GetMemberQuery): Member?
}

data class GetMemberQuery(
    val facilityId: FacilityId,
    val memberId: MemberId
)

@Service
@Transactional(readOnly = true)
class GetMemberService(
    private val memberRepository: MemberRepository
) : GetMemberUseCase {

    override fun execute(query: GetMemberQuery): Member? {
        val member = memberRepository.findById(query.memberId)

        // Ensure member belongs to the facility (multi-tenancy)
        return member?.takeIf { it.facilityId == query.facilityId }
    }
}
```

### 3. Domain Layer

**Responsibility:** Contain all business logic, entities, value objects, and domain services. **NO dependencies on other layers.**

```kotlin
/**
 * Member entity - core domain model.
 * Located in: domain/member/Member.kt
 */
data class Member(
    val id: MemberId,
    val facilityId: FacilityId,
    val personalInfo: PersonalInfo,
    val contactInfo: ContactInfo,
    val membershipStatus: MembershipStatus,
    val createdAt: Instant,
    val updatedAt: Instant
) {
    // Business logic methods here (shown earlier)
}

/**
 * Value objects used by Member.
 */
data class PersonalInfo(
    val arabicName: String,
    val englishName: String,
    val nationalId: NationalId,
    val dateOfBirth: LocalDate,
    val gender: Gender
) {
    init {
        require(arabicName.isNotBlank()) { "Arabic name cannot be blank" }
        require(englishName.isNotBlank()) { "English name cannot be blank" }
    }

    fun age(): Int {
        return Period.between(dateOfBirth, LocalDate.now()).years
    }
}

data class ContactInfo(
    val email: Email,
    val phone: PhoneNumber,
    val address: Address? = null
)

@JvmInline
value class NationalId(val value: String) {
    init {
        require(isValid(value)) {
            "Invalid Saudi national ID: $value"
        }
    }

    companion object {
        private val NATIONAL_ID_REGEX = """^[12][0-9]{9}$""".toRegex()

        fun isValid(id: String): Boolean {
            return NATIONAL_ID_REGEX.matches(id)
        }
    }
}

@JvmInline
value class PhoneNumber(val value: String) {
    init {
        require(isValid(value)) {
            "Invalid phone number: $value"
        }
    }

    companion object {
        private val PHONE_REGEX = """^\+966[0-9]{9}$""".toRegex()

        fun isValid(phone: String): Boolean {
            return PHONE_REGEX.matches(phone)
        }
    }
}

enum class Gender {
    MALE,
    FEMALE
}

enum class MembershipStatus {
    PENDING,
    ACTIVE,
    SUSPENDED,
    EXPIRED,
    CANCELLED
}

/**
 * Domain exceptions.
 */
sealed class DomainException(
    message: String,
    val errorCode: String
) : RuntimeException(message)

class MemberNotFoundException(memberId: MemberId) : DomainException(
    message = "Member not found: ${memberId.value}",
    errorCode = "MEMBER_NOT_FOUND"
)

class DuplicateEmailException(email: Email) : DomainException(
    message = "Email already exists: ${email.value}",
    errorCode = "DUPLICATE_EMAIL"
)

class FacilityNotFoundException(facilityId: FacilityId) : DomainException(
    message = "Facility not found: ${facilityId.value}",
    errorCode = "FACILITY_NOT_FOUND"
)
```

### 4. Infrastructure Layer

**Responsibility:** Implement technical capabilities - database access, external services, messaging, etc.

```kotlin
/**
 * JPA entity for persistence (separate from domain entity).
 */
@Entity
@Table(
    name = "members",
    indexes = [
        Index(name = "idx_member_facility", columnList = "facility_id"),
        Index(name = "idx_member_email", columnList = "email", unique = true)
    ]
)
class MemberJpaEntity(
    @Id
    @Column(name = "id", nullable = false, length = 36)
    val id: String,

    @Column(name = "facility_id", nullable = false, length = 36)
    val facilityId: String,

    @Column(name = "arabic_name", nullable = false, length = 255)
    val arabicName: String,

    @Column(name = "english_name", nullable = false, length = 255)
    val englishName: String,

    @Column(name = "email", nullable = false, unique = true, length = 255)
    val email: String,

    @Column(name = "phone", nullable = false, length = 20)
    val phone: String,

    @Column(name = "national_id", nullable = false, length = 10)
    val nationalId: String,

    @Column(name = "date_of_birth", nullable = false)
    val dateOfBirth: LocalDate,

    @Enumerated(EnumType.STRING)
    @Column(name = "gender", nullable = false, length = 10)
    val gender: Gender,

    @Enumerated(EnumType.STRING)
    @Column(name = "membership_status", nullable = false, length = 20)
    val membershipStatus: MembershipStatus,

    @Column(name = "created_at", nullable = false)
    val createdAt: Instant,

    @Column(name = "updated_at", nullable = false)
    val updatedAt: Instant
)

/**
 * Spring Data JPA repository interface.
 */
interface MemberJpaRepository : JpaRepository<MemberJpaEntity, String> {
    fun findByFacilityId(facilityId: String): List<MemberJpaEntity>
    fun findByEmail(email: String): MemberJpaEntity?
    fun existsByEmail(email: String): Boolean
}

/**
 * Repository adapter - implements domain repository using JPA.
 */
@Repository
class JpaMemberRepository(
    private val jpaRepository: MemberJpaRepository
) : MemberRepository {

    override fun save(member: Member): Member {
        val entity = member.toJpaEntity()
        val saved = jpaRepository.save(entity)
        return saved.toDomainModel()
    }

    override fun findById(id: MemberId): Member? {
        return jpaRepository.findById(id.value)
            .map { it.toDomainModel() }
            .orElse(null)
    }

    override fun findByFacilityId(facilityId: FacilityId): List<Member> {
        return jpaRepository.findByFacilityId(facilityId.value)
            .map { it.toDomainModel() }
    }

    override fun findByEmail(email: Email): Member? {
        return jpaRepository.findByEmail(email.value)?.toDomainModel()
    }

    override fun existsByEmail(email: Email): Boolean {
        return jpaRepository.existsByEmail(email.value)
    }

    override fun delete(id: MemberId) {
        jpaRepository.deleteById(id.value)
    }
}

/**
 * Mappers between domain and persistence models.
 */
fun Member.toJpaEntity(): MemberJpaEntity {
    return MemberJpaEntity(
        id = id.value,
        facilityId = facilityId.value,
        arabicName = personalInfo.arabicName,
        englishName = personalInfo.englishName,
        email = contactInfo.email.value,
        phone = contactInfo.phone.value,
        nationalId = personalInfo.nationalId.value,
        dateOfBirth = personalInfo.dateOfBirth,
        gender = personalInfo.gender,
        membershipStatus = membershipStatus,
        createdAt = createdAt,
        updatedAt = updatedAt
    )
}

fun MemberJpaEntity.toDomainModel(): Member {
    return Member(
        id = MemberId(id),
        facilityId = FacilityId(facilityId),
        personalInfo = PersonalInfo(
            arabicName = arabicName,
            englishName = englishName,
            nationalId = NationalId(nationalId),
            dateOfBirth = dateOfBirth,
            gender = gender
        ),
        contactInfo = ContactInfo(
            email = Email(email),
            phone = PhoneNumber(phone)
        ),
        membershipStatus = membershipStatus,
        enrolledClasses = emptyList(), // Loaded separately if needed
        createdAt = createdAt,
        updatedAt = updatedAt
    )
}

/**
 * Database migration using Flyway.
 * File: src/main/resources/db/migration/V001__create_members_table.sql
 */
/*
CREATE TABLE members (
    id VARCHAR(36) PRIMARY KEY,
    facility_id VARCHAR(36) NOT NULL,
    arabic_name VARCHAR(255) NOT NULL,
    english_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL,
    national_id VARCHAR(10) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender VARCHAR(10) NOT NULL,
    membership_status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,

    CONSTRAINT fk_member_facility FOREIGN KEY (facility_id)
        REFERENCES facilities(id) ON DELETE CASCADE
);

CREATE INDEX idx_member_facility ON members(facility_id);
CREATE UNIQUE INDEX idx_member_email ON members(email);
CREATE INDEX idx_member_national_id ON members(national_id);
*/
```

---

## Event-Driven Architecture

### Architecture Overview

FitHub uses **Event-Driven Architecture (EDA)** to enable:

1. **Loose Coupling:** Services communicate through events, not direct calls
2. **Scalability:** Async processing allows horizontal scaling
3. **Audit Trail:** All important actions are recorded as events
4. **Integration:** Easy to add new features that react to existing events
5. **Resilience:** Event replay enables recovery from failures

### Two-Level Event System

```
┌──────────────────────────────────────────────────────────────┐
│                     LEVEL 1: SPRING EVENTS                   │
│                  (Internal, In-Process)                      │
│                                                              │
│  ┌──────────────┐                    ┌──────────────────┐   │
│  │   Publisher  │ ─── Event ────────>│  Event Listener  │   │
│  │   (Service)  │                    │   (Same JVM)     │   │
│  └──────────────┘                    └──────────────────┘   │
│                                                              │
│  Use Cases:                                                  │
│  - Same transaction events                                   │
│  - Cache invalidation                                        │
│  - Internal notifications                                    │
│  - Read model updates (CQRS)                                │
└──────────────────────────────────────────────────────────────┘
                            │
                            │ For events requiring:
                            │ - Durability
                            │ - Cross-service communication
                            │ - Guaranteed delivery
                            ▼
┌──────────────────────────────────────────────────────────────┐
│                     LEVEL 2: APACHE KAFKA                    │
│                  (External, Distributed)                     │
│                                                              │
│  ┌──────────────┐    Kafka     ┌───────────────────┐        │
│  │   Producer   │ ──► Topic ───►│    Consumer       │        │
│  │  (Service A) │              │   (Service B)     │        │
│  └──────────────┘              └───────────────────┘        │
│                                                              │
│  Use Cases:                                                  │
│  - Cross-service integration                                 │
│  - External system notifications                             │
│  - Event sourcing / CQRS                                    │
│  - Analytics and reporting                                   │
└──────────────────────────────────────────────────────────────┘
```

### Spring Events Implementation

**Internal, synchronous/asynchronous event handling within the application:**

```kotlin
/**
 * Domain event publisher interface.
 */
interface DomainEventPublisher {
    fun publish(event: DomainEvent)
    fun publishAll(events: List<DomainEvent>)
}

/**
 * Spring-based event publisher implementation.
 */
@Component
class SpringDomainEventPublisher(
    private val applicationEventPublisher: ApplicationEventPublisher
) : DomainEventPublisher {

    override fun publish(event: DomainEvent) {
        applicationEventPublisher.publishEvent(event)
    }

    override fun publishAll(events: List<DomainEvent>) {
        events.forEach { publish(it) }
    }
}

/**
 * Event listener for membership events.
 */
@Component
class MembershipEventListener(
    private val emailService: EmailService,
    private val smsService: SmsService,
    private val analyticsService: AnalyticsService
) {

    private val logger = LoggerFactory.getLogger(javaClass)

    /**
     * Send welcome email when member is created.
     * Runs asynchronously to not block the main transaction.
     */
    @Async
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleMemberCreated(event: MemberCreated) {
        logger.info("Handling MemberCreated event: ${event.memberId}")

        try {
            emailService.sendWelcomeEmail(
                recipientEmail = event.email,
                memberName = event.memberName
            )

            smsService.sendWelcomeSms(
                phoneNumber = event.phone,
                memberName = event.memberName
            )

            analyticsService.trackMemberCreated(event)
        } catch (e: Exception) {
            logger.error("Failed to handle MemberCreated event", e)
            // Don't throw - notification failure shouldn't fail the transaction
        }
    }

    /**
     * Activate access when membership is activated.
     * Runs synchronously to ensure access is granted immediately.
     */
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleMembershipActivated(event: MembershipActivated) {
        logger.info("Handling MembershipActivated event: ${event.membershipId}")

        // Grant access to facility systems
        facilityAccessService.grantAccess(
            memberId = event.memberId,
            facilityId = event.facilityId,
            accessLevel = event.membershipType.accessLevel()
        )

        // Send activation notification
        notificationService.sendMembershipActivatedNotification(event)
    }

    /**
     * Update read model when payment is processed (CQRS).
     */
    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handlePaymentProcessed(event: PaymentProcessed) {
        logger.info("Handling PaymentProcessed event: ${event.paymentId}")

        // Update denormalized read model for reporting
        paymentReadModelRepository.save(
            PaymentReadModel(
                paymentId = event.paymentId.value,
                memberId = event.memberId.value,
                amount = event.amount.amount,
                currency = event.amount.currency.code,
                paymentMethod = event.paymentMethod.name,
                transactionId = event.transactionId,
                processedAt = event.occurredAt
            )
        )
    }
}

/**
 * Enable async processing for @Async methods.
 */
@Configuration
@EnableAsync
class AsyncConfiguration {

    @Bean
    fun taskExecutor(): Executor {
        return ThreadPoolTaskExecutor().apply {
            corePoolSize = 10
            maxPoolSize = 50
            queueCapacity = 100
            setThreadNamePrefix("async-event-")
            initialize()
        }
    }
}
```

### Apache Kafka Implementation

**External, distributed event streaming for cross-service communication:**

```kotlin
/**
 * Kafka configuration.
 */
@Configuration
@EnableKafka
class KafkaConfiguration {

    @Value("\${spring.kafka.bootstrap-servers}")
    private lateinit var bootstrapServers: String

    @Bean
    fun producerFactory(): ProducerFactory<String, DomainEvent> {
        val config = mapOf(
            ProducerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,
            ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG to StringSerializer::class.java,
            ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG to JsonSerializer::class.java,
            ProducerConfig.ACKS_CONFIG to "all", // Wait for all replicas
            ProducerConfig.RETRIES_CONFIG to 3,
            ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG to true,
            JsonSerializer.ADD_TYPE_INFO_HEADERS to false
        )
        return DefaultKafkaProducerFactory(config)
    }

    @Bean
    fun kafkaTemplate(): KafkaTemplate<String, DomainEvent> {
        return KafkaTemplate(producerFactory())
    }

    @Bean
    fun consumerFactory(): ConsumerFactory<String, DomainEvent> {
        val config = mapOf(
            ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG to bootstrapServers,
            ConsumerConfig.GROUP_ID_CONFIG to "fithub-api",
            ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG to StringDeserializer::class.java,
            ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG to JsonDeserializer::class.java,
            ConsumerConfig.AUTO_OFFSET_RESET_CONFIG to "earliest",
            JsonDeserializer.TRUSTED_PACKAGES to "com.fithub.domain.events",
            JsonDeserializer.USE_TYPE_INFO_HEADERS to false
        )
        return DefaultKafkaConsumerFactory(config)
    }

    @Bean
    fun kafkaListenerContainerFactory(): ConcurrentKafkaListenerContainerFactory<String, DomainEvent> {
        val factory = ConcurrentKafkaListenerContainerFactory<String, DomainEvent>()
        factory.consumerFactory = consumerFactory()
        factory.setConcurrency(3) // 3 consumer threads
        factory.containerProperties.ackMode = ContainerProperties.AckMode.RECORD
        return factory
    }
}

/**
 * Kafka topics configuration.
 */
object KafkaTopics {
    const val MEMBERSHIP_EVENTS = "fithub.membership.events"
    const val PAYMENT_EVENTS = "fithub.payment.events"
    const val CLASS_EVENTS = "fithub.class.events"
    const val NOTIFICATION_EVENTS = "fithub.notification.events"
}

/**
 * Kafka event publisher.
 */
@Component
class KafkaEventPublisher(
    private val kafkaTemplate: KafkaTemplate<String, DomainEvent>
) {

    private val logger = LoggerFactory.getLogger(javaClass)

    fun publish(topic: String, event: DomainEvent) {
        logger.info("Publishing event to Kafka: ${event.eventType} -> $topic")

        val future = kafkaTemplate.send(
            topic,
            event.aggregateId, // Use aggregate ID as partition key
            event
        )

        future.whenComplete { result, ex ->
            if (ex == null) {
                logger.info(
                    "Event published successfully: ${event.eventType}, " +
                    "partition=${result?.recordMetadata?.partition()}, " +
                    "offset=${result?.recordMetadata?.offset()}"
                )
            } else {
                logger.error("Failed to publish event: ${event.eventType}", ex)
            }
        }
    }
}

/**
 * Bridge Spring events to Kafka for important business events.
 */
@Component
class EventBridgeListener(
    private val kafkaEventPublisher: KafkaEventPublisher
) {

    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun bridgeMembershipEvents(event: MembershipActivated) {
        // Publish to Kafka for external consumers
        kafkaEventPublisher.publish(KafkaTopics.MEMBERSHIP_EVENTS, event)
    }

    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun bridgePaymentEvents(event: PaymentProcessed) {
        kafkaEventPublisher.publish(KafkaTopics.PAYMENT_EVENTS, event)
    }
}

/**
 * Kafka consumer for analytics service.
 */
@Component
class AnalyticsEventConsumer(
    private val analyticsRepository: AnalyticsRepository
) {

    private val logger = LoggerFactory.getLogger(javaClass)

    @KafkaListener(
        topics = [KafkaTopics.MEMBERSHIP_EVENTS, KafkaTopics.PAYMENT_EVENTS],
        groupId = "analytics-service"
    )
    fun consumeEvents(event: DomainEvent) {
        logger.info("Consumed event for analytics: ${event.eventType}")

        when (event) {
            is MembershipActivated -> trackMembershipActivation(event)
            is PaymentProcessed -> trackPayment(event)
            else -> logger.debug("Ignoring event type: ${event.eventType}")
        }
    }

    private fun trackMembershipActivation(event: MembershipActivated) {
        analyticsRepository.recordMembershipActivation(
            facilityId = event.facilityId,
            membershipType = event.membershipType,
            timestamp = event.occurredAt
        )
    }

    private fun trackPayment(event: PaymentProcessed) {
        analyticsRepository.recordRevenue(
            amount = event.amount,
            paymentMethod = event.paymentMethod,
            timestamp = event.occurredAt
        )
    }
}
```

### Event Sourcing (Advanced Pattern)

For critical aggregates like payments, FitHub can use **Event Sourcing:**

```kotlin
/**
 * Event-sourced Payment aggregate.
 * State is reconstructed from event history.
 */
class Payment private constructor(
    val id: PaymentId,
    private val events: MutableList<PaymentEvent> = mutableListOf()
) {
    var status: PaymentStatus = PaymentStatus.PENDING
        private set

    var amount: Money = Money(BigDecimal.ZERO)
        private set

    var memberId: MemberId? = null
        private set

    companion object {
        /**
         * Reconstruct payment from event history.
         */
        fun fromHistory(id: PaymentId, history: List<PaymentEvent>): Payment {
            val payment = Payment(id)
            history.forEach { payment.apply(it) }
            return payment
        }

        /**
         * Create new payment.
         */
        fun create(id: PaymentId, memberId: MemberId, amount: Money): Payment {
            val payment = Payment(id)
            payment.processEvent(
                PaymentInitiated(
                    paymentId = id,
                    memberId = memberId,
                    amount = amount,
                    occurredAt = Instant.now()
                )
            )
            return payment
        }
    }

    fun authorize(): Payment {
        processEvent(
            PaymentAuthorized(
                paymentId = id,
                occurredAt = Instant.now()
            )
        )
        return this
    }

    fun complete(transactionId: String): Payment {
        processEvent(
            PaymentCompleted(
                paymentId = id,
                transactionId = transactionId,
                occurredAt = Instant.now()
            )
        )
        return this
    }

    fun fail(reason: String): Payment {
        processEvent(
            PaymentFailed(
                paymentId = id,
                reason = reason,
                occurredAt = Instant.now()
            )
        )
        return this
    }

    /**
     * Get uncommitted events for publishing.
     */
    fun uncommittedEvents(): List<PaymentEvent> = events.toList()

    /**
     * Mark events as committed.
     */
    fun markEventsAsCommitted() {
        events.clear()
    }

    private fun processEvent(event: PaymentEvent) {
        apply(event)
        events.add(event)
    }

    /**
     * Apply event to update state.
     */
    private fun apply(event: PaymentEvent) {
        when (event) {
            is PaymentInitiated -> {
                memberId = event.memberId
                amount = event.amount
                status = PaymentStatus.PENDING
            }
            is PaymentAuthorized -> {
                status = PaymentStatus.AUTHORIZED
            }
            is PaymentCompleted -> {
                status = PaymentStatus.COMPLETED
            }
            is PaymentFailed -> {
                status = PaymentStatus.FAILED
            }
        }
    }
}

/**
 * Event store for persisting events.
 */
interface EventStore {
    fun saveEvents(aggregateId: String, events: List<DomainEvent>, expectedVersion: Int)
    fun getEvents(aggregateId: String): List<DomainEvent>
}

@Repository
class JpaEventStore(
    private val eventStoreRepository: EventStoreJpaRepository
) : EventStore {

    override fun saveEvents(
        aggregateId: String,
        events: List<DomainEvent>,
        expectedVersion: Int
    ) {
        events.forEachIndexed { index, event ->
            val entity = EventStoreEntry(
                aggregateId = aggregateId,
                aggregateType = "Payment",
                version = expectedVersion + index + 1,
                eventType = event.eventType,
                eventData = serializeEvent(event),
                occurredAt = event.occurredAt
            )
            eventStoreRepository.save(entity)
        }
    }

    override fun getEvents(aggregateId: String): List<DomainEvent> {
        return eventStoreRepository.findByAggregateIdOrderByVersionAsc(aggregateId)
            .map { deserializeEvent(it.eventData, it.eventType) }
    }

    private fun serializeEvent(event: DomainEvent): String {
        return objectMapper.writeValueAsString(event)
    }

    private fun deserializeEvent(data: String, type: String): DomainEvent {
        val clazz = Class.forName("com.fithub.domain.events.$type")
        return objectMapper.readValue(data, clazz) as DomainEvent
    }
}
```

---

## CQRS Pattern

### Overview

**CQRS (Command Query Responsibility Segregation)** separates read and write operations:

- **Commands:** Modify state (Create, Update, Delete)
- **Queries:** Read state (Get, List, Search)

### Why CQRS?

1. **Optimized Read Models:** Denormalized views for fast queries
2. **Scalability:** Scale reads and writes independently
3. **Simplicity:** Queries don't need complex domain logic
4. **Performance:** Read models can use different databases (PostgreSQL for writes, Elasticsearch for search)
5. **Flexibility:** Multiple read models for different use cases

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                       WRITE SIDE (Commands)                  │
│                                                              │
│  REST API ──► Command ──► Use Case ──► Domain Model ──►     │
│                                            │                 │
│                                            ▼                 │
│                                    PostgreSQL (Write DB)     │
│                                            │                 │
│                                            ▼                 │
│                                      Domain Events           │
└──────────────────────┬───────────────────────────────────────┘
                       │
                       │ Events published
                       ▼
┌──────────────────────────────────────────────────────────────┐
│                       READ SIDE (Queries)                    │
│                                                              │
│  Domain Events ──► Event Handler ──► Update Read Models     │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐  │
│  │  PostgreSQL    │  │ Elasticsearch  │  │    Redis     │  │
│  │  Read Models   │  │  Full-text     │  │    Cache     │  │
│  └────────────────┘  └────────────────┘  └──────────────┘  │
│          │                   │                   │          │
│          └───────────────────┴───────────────────┘          │
│                              │                              │
│  REST API ◄──── Query ◄──────┘                              │
└──────────────────────────────────────────────────────────────┘
```

### Implementation

#### Write Side (Commands)

```kotlin
/**
 * Command for creating a member.
 */
data class CreateMemberCommand(
    val facilityId: FacilityId,
    val personalInfo: PersonalInfo,
    val contactInfo: ContactInfo
)

/**
 * Command handler (write side).
 */
@Service
@Transactional
class CreateMemberCommandHandler(
    private val memberRepository: MemberRepository,
    private val eventPublisher: DomainEventPublisher
) {
    fun handle(command: CreateMemberCommand): MemberId {
        // 1. Create domain entity
        val member = Member.create(
            facilityId = command.facilityId,
            personalInfo = command.personalInfo,
            contactInfo = command.contactInfo
        )

        // 2. Save to write database
        memberRepository.save(member)

        // 3. Publish event for read model updates
        eventPublisher.publish(
            MemberCreated(
                aggregateId = member.id.value,
                memberId = member.id,
                facilityId = member.facilityId,
                personalInfo = member.personalInfo,
                contactInfo = member.contactInfo,
                occurredAt = Instant.now()
            )
        )

        return member.id
    }
}
```

#### Read Side (Queries)

```kotlin
/**
 * Denormalized read model for member list view.
 * Optimized for fast queries with pre-joined data.
 */
@Entity
@Table(name = "member_list_view")
data class MemberListReadModel(
    @Id
    val memberId: String,
    val facilityId: String,
    val arabicName: String,
    val englishName: String,
    val email: String,
    val phone: String,
    val membershipStatus: String,
    val membershipType: String?,
    val membershipExpiryDate: LocalDate?,
    val lastVisitDate: Instant?,
    val totalVisits: Int,
    val outstandingBalance: BigDecimal,
    val createdAt: Instant,
    val updatedAt: Instant
)

/**
 * Query for listing members with filters.
 */
data class ListMembersQuery(
    val facilityId: FacilityId,
    val status: MembershipStatus? = null,
    val searchTerm: String? = null,
    val page: Int = 0,
    val size: Int = 20
)

/**
 * Query handler (read side).
 */
@Service
@Transactional(readOnly = true)
class ListMembersQueryHandler(
    private val memberListRepository: MemberListReadModelRepository
) {
    fun handle(query: ListMembersQuery): Page<MemberListReadModel> {
        // Query the denormalized read model
        return if (query.searchTerm != null) {
            memberListRepository.searchMembers(
                facilityId = query.facilityId.value,
                searchTerm = query.searchTerm,
                pageable = PageRequest.of(query.page, query.size)
            )
        } else {
            memberListRepository.findByFacilityId(
                facilityId = query.facilityId.value,
                status = query.status?.name,
                pageable = PageRequest.of(query.page, query.size)
            )
        }
    }
}

/**
 * Read model repository with optimized queries.
 */
interface MemberListReadModelRepository : JpaRepository<MemberListReadModel, String> {

    fun findByFacilityId(
        facilityId: String,
        status: String?,
        pageable: Pageable
    ): Page<MemberListReadModel>

    @Query("""
        SELECT m FROM MemberListReadModel m
        WHERE m.facilityId = :facilityId
        AND (m.arabicName LIKE %:searchTerm%
             OR m.englishName LIKE %:searchTerm%
             OR m.email LIKE %:searchTerm%
             OR m.phone LIKE %:searchTerm%)
    """)
    fun searchMembers(
        facilityId: String,
        searchTerm: String,
        pageable: Pageable
    ): Page<MemberListReadModel>
}

/**
 * Event handler to update read models.
 */
@Component
class MemberReadModelProjection(
    private val memberListRepository: MemberListReadModelRepository
) {

    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleMemberCreated(event: MemberCreated) {
        val readModel = MemberListReadModel(
            memberId = event.memberId.value,
            facilityId = event.facilityId.value,
            arabicName = event.personalInfo.arabicName,
            englishName = event.personalInfo.englishName,
            email = event.contactInfo.email.value,
            phone = event.contactInfo.phone.value,
            membershipStatus = MembershipStatus.PENDING.name,
            membershipType = null,
            membershipExpiryDate = null,
            lastVisitDate = null,
            totalVisits = 0,
            outstandingBalance = BigDecimal.ZERO,
            createdAt = event.occurredAt,
            updatedAt = event.occurredAt
        )

        memberListRepository.save(readModel)
    }

    @EventListener
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleMembershipActivated(event: MembershipActivated) {
        memberListRepository.findById(event.memberId.value).ifPresent { readModel ->
            val updated = readModel.copy(
                membershipStatus = MembershipStatus.ACTIVE.name,
                membershipType = event.membershipType.name,
                membershipExpiryDate = event.endDate,
                updatedAt = event.occurredAt
            )
            memberListRepository.save(updated)
        }
    }
}
```

### CQRS with Elasticsearch (Advanced)

For full-text search and complex queries:

```kotlin
/**
 * Elasticsearch document for member search.
 */
@Document(indexName = "members")
data class MemberSearchDocument(
    @Id
    val id: String,
    @Field(type = FieldType.Keyword)
    val facilityId: String,
    @Field(type = FieldType.Text, analyzer = "arabic")
    val arabicName: String,
    @Field(type = FieldType.Text, analyzer = "standard")
    val englishName: String,
    @Field(type = FieldType.Keyword)
    val email: String,
    @Field(type = FieldType.Keyword)
    val phone: String,
    @Field(type = FieldType.Keyword)
    val membershipStatus: String,
    @Field(type = FieldType.Date)
    val createdAt: Instant
)

/**
 * Elasticsearch repository.
 */
interface MemberSearchRepository : ElasticsearchRepository<MemberSearchDocument, String> {

    fun findByFacilityIdAndMembershipStatus(
        facilityId: String,
        membershipStatus: String
    ): List<MemberSearchDocument>

    @Query("""
        {
          "bool": {
            "must": [
              { "term": { "facilityId": "?0" } },
              {
                "multi_match": {
                  "query": "?1",
                  "fields": ["arabicName^2", "englishName^2", "email", "phone"]
                }
              }
            ]
          }
        }
    """)
    fun searchMembers(facilityId: String, searchTerm: String): List<MemberSearchDocument>
}

/**
 * Event handler to update Elasticsearch index.
 */
@Component
class MemberSearchProjection(
    private val memberSearchRepository: MemberSearchRepository
) {

    @EventListener
    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    fun handleMemberCreated(event: MemberCreated) {
        val document = MemberSearchDocument(
            id = event.memberId.value,
            facilityId = event.facilityId.value,
            arabicName = event.personalInfo.arabicName,
            englishName = event.personalInfo.englishName,
            email = event.contactInfo.email.value,
            phone = event.contactInfo.phone.value,
            membershipStatus = MembershipStatus.PENDING.name,
            createdAt = event.occurredAt
        )

        memberSearchRepository.save(document)
    }
}
```

---

## Project Structure

```
fithub-backend/
├── build.gradle.kts                 # Gradle build configuration
├── settings.gradle.kts              # Multi-module settings
├── gradle/
│   └── libs.versions.toml          # Centralized dependency versions
│
├── src/
│   ├── main/
│   │   ├── kotlin/
│   │   │   └── com/fithub/
│   │   │       │
│   │   │       ├── FitHubApplication.kt  # Main application entry point
│   │   │       │
│   │   │       ├── api/                  # PRESENTATION LAYER
│   │   │       │   ├── rest/
│   │   │       │   │   ├── MemberController.kt
│   │   │       │   │   ├── MembershipController.kt
│   │   │       │   │   ├── ClassController.kt
│   │   │       │   │   ├── PaymentController.kt
│   │   │       │   │   └── FacilityController.kt
│   │   │       │   ├── dto/
│   │   │       │   │   ├── request/
│   │   │       │   │   │   ├── CreateMemberRequest.kt
│   │   │       │   │   │   └── UpdateMemberRequest.kt
│   │   │       │   │   └── response/
│   │   │       │   │       ├── MemberResponse.kt
│   │   │       │   │       └── ErrorResponse.kt
│   │   │       │   └── exception/
│   │   │       │       └── GlobalExceptionHandler.kt
│   │   │       │
│   │   │       ├── application/          # APPLICATION LAYER
│   │   │       │   ├── usecase/
│   │   │       │   │   ├── member/
│   │   │       │   │   │   ├── CreateMemberUseCase.kt
│   │   │       │   │   │   ├── CreateMemberService.kt
│   │   │       │   │   │   ├── GetMemberUseCase.kt
│   │   │       │   │   │   └── GetMemberService.kt
│   │   │       │   │   ├── membership/
│   │   │       │   │   ├── payment/
│   │   │       │   │   └── class/
│   │   │       │   ├── command/
│   │   │       │   │   ├── CreateMemberCommand.kt
│   │   │       │   │   └── UpdateMemberCommand.kt
│   │   │       │   ├── query/
│   │   │       │   │   ├── GetMemberQuery.kt
│   │   │       │   │   └── ListMembersQuery.kt
│   │   │       │   └── mapper/
│   │   │       │       └── MemberMapper.kt
│   │   │       │
│   │   │       ├── domain/               # DOMAIN LAYER (Core)
│   │   │       │   ├── member/
│   │   │       │   │   ├── Member.kt          # Entity
│   │   │       │   │   ├── MemberId.kt        # Value Object
│   │   │       │   │   ├── PersonalInfo.kt    # Value Object
│   │   │       │   │   ├── MemberRepository.kt # Port
│   │   │       │   │   └── MembershipStatus.kt
│   │   │       │   ├── membership/
│   │   │       │   │   ├── Membership.kt
│   │   │       │   │   ├── FamilyAccount.kt   # Aggregate Root
│   │   │       │   │   ├── MembershipType.kt
│   │   │       │   │   └── BillingPeriod.kt
│   │   │       │   ├── facility/
│   │   │       │   │   ├── Facility.kt
│   │   │       │   │   ├── FacilityId.kt
│   │   │       │   │   ├── Location.kt
│   │   │       │   │   └── GenderSegregation.kt
│   │   │       │   ├── class/
│   │   │       │   │   ├── Class.kt
│   │   │       │   │   ├── ClassSession.kt
│   │   │       │   │   ├── Schedule.kt
│   │   │       │   │   └── Attendance.kt
│   │   │       │   ├── payment/
│   │   │       │   │   ├── Payment.kt
│   │   │       │   │   ├── Invoice.kt
│   │   │       │   │   ├── Transaction.kt
│   │   │       │   │   └── PaymentMethod.kt
│   │   │       │   ├── shared/
│   │   │       │   │   ├── vo/              # Shared Value Objects
│   │   │       │   │   │   ├── Money.kt
│   │   │       │   │   │   ├── Email.kt
│   │   │       │   │   │   ├── PhoneNumber.kt
│   │   │       │   │   │   ├── HijriDate.kt
│   │   │       │   │   │   └── PrayerTime.kt
│   │   │       │   │   ├── service/         # Domain Services
│   │   │       │   │   │   ├── PricingService.kt
│   │   │       │   │   │   └── SchedulingService.kt
│   │   │       │   │   └── event/           # Domain Events
│   │   │       │   │       ├── DomainEvent.kt
│   │   │       │   │       ├── MemberCreated.kt
│   │   │       │   │       ├── MembershipActivated.kt
│   │   │       │   │       └── PaymentProcessed.kt
│   │   │       │   └── exception/
│   │   │       │       └── DomainException.kt
│   │   │       │
│   │   │       └── infrastructure/       # INFRASTRUCTURE LAYER
│   │   │           ├── persistence/
│   │   │           │   ├── jpa/
│   │   │           │   │   ├── entity/
│   │   │           │   │   │   ├── MemberJpaEntity.kt
│   │   │           │   │   │   ├── MembershipJpaEntity.kt
│   │   │           │   │   │   └── PaymentJpaEntity.kt
│   │   │           │   │   ├── repository/
│   │   │           │   │   │   ├── MemberJpaRepository.kt
│   │   │           │   │   │   ├── JpaMemberRepository.kt  # Adapter
│   │   │           │   │   │   └── MembershipJpaRepository.kt
│   │   │           │   │   └── mapper/
│   │   │           │   │       └── MemberJpaMapper.kt
│   │   │           │   ├── elasticsearch/
│   │   │           │   │   ├── document/
│   │   │           │   │   │   └── MemberSearchDocument.kt
│   │   │           │   │   └── repository/
│   │   │           │   │       └── MemberSearchRepository.kt
│   │   │           │   └── redis/
│   │   │           │       └── MemberCacheRepository.kt
│   │   │           │
│   │   │           ├── messaging/
│   │   │           │   ├── kafka/
│   │   │           │   │   ├── KafkaEventPublisher.kt
│   │   │           │   │   ├── KafkaEventConsumer.kt
│   │   │           │   │   └── KafkaConfiguration.kt
│   │   │           │   └── spring/
│   │   │           │       ├── SpringEventPublisher.kt
│   │   │           │       └── SpringEventListener.kt
│   │   │           │
│   │   │           ├── external/
│   │   │           │   ├── payment/
│   │   │           │   │   ├── StripePaymentGateway.kt
│   │   │           │   │   └── MadaPaymentGateway.kt
│   │   │           │   ├── notification/
│   │   │           │   │   ├── TwilioSmsService.kt
│   │   │           │   │   └── SendGridEmailService.kt
│   │   │           │   ├── zatca/
│   │   │           │   │   └── ZatcaEInvoicingClient.kt
│   │   │           │   └── calendar/
│   │   │           │       ├── HijriCalendarService.kt
│   │   │           │       └── PrayerTimeProvider.kt
│   │   │           │
│   │   │           ├── security/
│   │   │           │   ├── JwtAuthenticationFilter.kt
│   │   │           │   ├── MultiTenantFilter.kt
│   │   │           │   └── SecurityConfiguration.kt
│   │   │           │
│   │   │           └── config/
│   │   │               ├── DatabaseConfiguration.kt
│   │   │               ├── KafkaConfiguration.kt
│   │   │               ├── RedisConfiguration.kt
│   │   │               ├── OpenApiConfiguration.kt
│   │   │               └── AsyncConfiguration.kt
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       ├── application-dev.yml
│   │       ├── application-prod.yml
│   │       ├── db/
│   │       │   └── migration/
│   │       │       ├── V001__create_facilities_table.sql
│   │       │       ├── V002__create_members_table.sql
│   │       │       ├── V003__create_memberships_table.sql
│   │       │       └── V004__create_payments_table.sql
│   │       └── messages/
│   │           ├── messages_en.properties
│   │           └── messages_ar.properties
│   │
│   └── test/
│       ├── kotlin/
│       │   └── com/fithub/
│       │       ├── domain/              # Domain unit tests
│       │       │   ├── MemberTest.kt
│       │       │   ├── FamilyAccountTest.kt
│       │       │   └── PricingServiceTest.kt
│       │       ├── application/         # Use case tests
│       │       │   └── CreateMemberServiceTest.kt
│       │       ├── api/                 # Integration tests
│       │       │   └── MemberControllerTest.kt
│       │       └── architecture/        # Architecture tests
│       │           └── ArchitectureTest.kt
│       └── resources/
│           └── application-test.yml
│
└── docker/
    ├── docker-compose.yml
    ├── Dockerfile
    └── postgres/
        └── init.sql
```

---

## Data Architecture

### Database Schema Design

**PostgreSQL** is used for the primary transactional database.

#### Multi-Tenancy Strategy

FitHub uses **Schema-per-Tenant** approach:

```sql
-- Each facility gets its own schema
CREATE SCHEMA facility_abc123;
CREATE SCHEMA facility_xyz789;

-- Tables created in each schema
CREATE TABLE facility_abc123.members (...);
CREATE TABLE facility_xyz789.members (...);
```

**Rationale:**
- **Data Isolation:** Complete separation between tenants
- **Security:** PostgreSQL row-level security enforces access
- **Scalability:** Easy to move tenant schemas to different databases
- **Compliance:** Meets PDPL requirements for data segregation

#### Core Tables

```sql
-- Facilities table (shared schema)
CREATE TABLE public.facilities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_arabic VARCHAR(255) NOT NULL,
    name_english VARCHAR(255) NOT NULL,
    license_number VARCHAR(50) NOT NULL UNIQUE,
    owner_name VARCHAR(255) NOT NULL,
    owner_email VARCHAR(255) NOT NULL,
    subscription_tier VARCHAR(20) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Members table (per-facility schema)
CREATE TABLE {facility_schema}.members (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    arabic_name VARCHAR(255) NOT NULL,
    english_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    phone VARCHAR(20) NOT NULL,
    national_id VARCHAR(10) NOT NULL,
    date_of_birth DATE NOT NULL,
    gender VARCHAR(10) NOT NULL,
    membership_status VARCHAR(20) NOT NULL,
    family_account_id UUID NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_family_account FOREIGN KEY (family_account_id)
        REFERENCES {facility_schema}.family_accounts(id)
);

CREATE INDEX idx_members_email ON {facility_schema}.members(email);
CREATE INDEX idx_members_national_id ON {facility_schema}.members(national_id);
CREATE INDEX idx_members_family_account ON {facility_schema}.members(family_account_id);

-- Family accounts
CREATE TABLE {facility_schema}.family_accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    primary_member_id UUID NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_primary_member FOREIGN KEY (primary_member_id)
        REFERENCES {facility_schema}.members(id)
);

-- Memberships
CREATE TABLE {facility_schema}.memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL,
    membership_type VARCHAR(20) NOT NULL,
    billing_period VARCHAR(20) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL,
    price_amount DECIMAL(10, 2) NOT NULL,
    price_currency VARCHAR(3) NOT NULL DEFAULT 'SAR',
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_member FOREIGN KEY (member_id)
        REFERENCES {facility_schema}.members(id) ON DELETE CASCADE
);

CREATE INDEX idx_memberships_member ON {facility_schema}.memberships(member_id);
CREATE INDEX idx_memberships_dates ON {facility_schema}.memberships(start_date, end_date);

-- Payments
CREATE TABLE {facility_schema}.payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    member_id UUID NOT NULL,
    membership_id UUID NULL,
    amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL DEFAULT 'SAR',
    payment_method VARCHAR(20) NOT NULL,
    transaction_id VARCHAR(255) NULL,
    status VARCHAR(20) NOT NULL,
    processed_at TIMESTAMP NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_payment_member FOREIGN KEY (member_id)
        REFERENCES {facility_schema}.members(id),
    CONSTRAINT fk_payment_membership FOREIGN KEY (membership_id)
        REFERENCES {facility_schema}.memberships(id)
);

CREATE INDEX idx_payments_member ON {facility_schema}.payments(member_id);
CREATE INDEX idx_payments_status ON {facility_schema}.payments(status);

-- Classes
CREATE TABLE {facility_schema}.classes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name_arabic VARCHAR(255) NOT NULL,
    name_english VARCHAR(255) NOT NULL,
    description TEXT,
    instructor_id UUID NOT NULL,
    max_capacity INT NOT NULL,
    gender_restriction VARCHAR(10) NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Class sessions
CREATE TABLE {facility_schema}.class_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    class_id UUID NOT NULL,
    instructor_id UUID NOT NULL,
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP NOT NULL,
    status VARCHAR(20) NOT NULL,
    enrolled_count INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_session_class FOREIGN KEY (class_id)
        REFERENCES {facility_schema}.classes(id) ON DELETE CASCADE
);

CREATE INDEX idx_class_sessions_class ON {facility_schema}.class_sessions(class_id);
CREATE INDEX idx_class_sessions_time ON {facility_schema}.class_sessions(start_time, end_time);

-- Event store (for event sourcing)
CREATE TABLE public.event_store (
    id BIGSERIAL PRIMARY KEY,
    aggregate_id VARCHAR(36) NOT NULL,
    aggregate_type VARCHAR(50) NOT NULL,
    version INT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    occurred_at TIMESTAMP NOT NULL,

    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_event_store_aggregate ON public.event_store(aggregate_id, version);
```

### Caching Strategy (Redis)

```kotlin
@Configuration
@EnableCaching
class CacheConfiguration {

    @Bean
    fun cacheManager(redisConnectionFactory: RedisConnectionFactory): CacheManager {
        val config = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    StringRedisSerializer()
                )
            )
            .serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(
                    GenericJackson2JsonRedisSerializer()
                )
            )

        return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(config)
            .withCacheConfiguration(
                "members",
                config.entryTtl(Duration.ofHours(1))
            )
            .withCacheConfiguration(
                "facilities",
                config.entryTtl(Duration.ofHours(24))
            )
            .build()
    }
}

// Usage in service
@Service
class GetMemberService(
    private val memberRepository: MemberRepository
) : GetMemberUseCase {

    @Cacheable(value = ["members"], key = "#query.memberId.value")
    override fun execute(query: GetMemberQuery): Member? {
        return memberRepository.findById(query.memberId)
    }
}
```

---

## Security Architecture

### Multi-Tenant Isolation

```kotlin
/**
 * Extracts facility ID from request and sets tenant context.
 */
@Component
class MultiTenantFilter : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val facilityId = request.getHeader("X-Facility-Id")
            ?: throw MissingTenantException()

        try {
            TenantContext.setCurrentTenant(facilityId)
            filterChain.doFilter(request, response)
        } finally {
            TenantContext.clear()
        }
    }
}

/**
 * Thread-local tenant context.
 */
object TenantContext {
    private val currentTenant = ThreadLocal<String>()

    fun setCurrentTenant(tenantId: String) {
        currentTenant.set(tenantId)
    }

    fun getCurrentTenant(): String {
        return currentTenant.get() ?: throw NoTenantContextException()
    }

    fun clear() {
        currentTenant.remove()
    }
}

/**
 * Tenant-aware data source routing.
 */
class MultiTenantDataSource : AbstractRoutingDataSource() {

    override fun determineCurrentLookupKey(): Any {
        return TenantContext.getCurrentTenant()
    }
}
```

### Authentication & Authorization

```kotlin
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
class SecurityConfiguration {

    @Bean
    fun securityFilterChain(http: HttpSecurity): SecurityFilterChain {
        http
            .csrf { it.disable() }
            .authorizeHttpRequests { auth ->
                auth
                    .requestMatchers("/api/v1/public/**").permitAll()
                    .requestMatchers("/actuator/health").permitAll()
                    .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                    .requestMatchers("/api/v1/**").authenticated()
            }
            .oauth2ResourceServer { oauth2 ->
                oauth2.jwt { jwt ->
                    jwt.jwtAuthenticationConverter(jwtAuthenticationConverter())
                }
            }
            .sessionManagement { session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            }

        return http.build()
    }

    private fun jwtAuthenticationConverter(): JwtAuthenticationConverter {
        val converter = JwtAuthenticationConverter()
        converter.setJwtGrantedAuthoritiesConverter(
            JwtGrantedAuthoritiesConverter().apply {
                setAuthorityPrefix("ROLE_")
                setAuthoritiesClaimName("roles")
            }
        )
        return converter
    }
}
```

---

## Architectural Decision Records

### ADR-001: Kotlin over Java

**Decision:** Use Kotlin as the primary backend language.

**Rationale:**
- **Null Safety:** Prevents NPEs at compile time
- **Conciseness:** 40% less code than Java
- **Coroutines:** Superior async programming model
- **Data Classes:** Perfect for DDD value objects
- **Interoperability:** 100% compatible with Java libraries
- **Mobile Sharing:** Enable Kotlin Multiplatform for iOS/Android

**Consequences:**
- Team must learn Kotlin (2-week ramp-up)
- Excellent IDE support in IntelliJ IDEA
- Growing ecosystem and community

---

### ADR-002: Hexagonal Architecture

**Decision:** Implement Hexagonal Architecture (Ports & Adapters).

**Rationale:**
- **Testability:** Business logic testable without infrastructure
- **Flexibility:** Easy to swap databases, message brokers, frameworks
- **Technology Independence:** Domain layer has zero framework dependencies
- **Clarity:** Clear separation of concerns

**Consequences:**
- More interfaces and mappers
- Steeper learning curve for junior developers
- Higher initial development time, but faster feature development later

---

### ADR-003: CQRS for Complex Operations

**Decision:** Use CQRS pattern for member/payment queries.

**Rationale:**
- **Performance:** Optimized read models for reporting
- **Scalability:** Scale reads and writes independently
- **Simplicity:** Queries don't need complex domain logic
- **Flexibility:** Multiple read models for different use cases

**Consequences:**
- Eventual consistency between write and read models
- More code (separate read/write models)
- Need for event handlers to update read models

---

### ADR-004: Event-Driven Architecture

**Decision:** Use Spring Events (internal) + Apache Kafka (external).

**Rationale:**
- **Decoupling:** Services communicate through events
- **Scalability:** Async processing enables horizontal scaling
- **Audit Trail:** All business events are recorded
- **Integration:** Easy to add analytics, notifications, etc.

**Consequences:**
- Debugging is harder (distributed traces needed)
- Eventual consistency model
- Need Kafka infrastructure

---

### ADR-005: PostgreSQL for Primary Database

**Decision:** Use PostgreSQL as the primary transactional database.

**Rationale:**
- **ACID Compliance:** Full transactional support
- **JSON Support:** JSONB for flexible data structures
- **Performance:** Excellent for OLTP workloads
- **Multi-Tenancy:** Schema isolation support
- **Maturity:** Battle-tested, excellent tooling

**Consequences:**
- Need PostgreSQL expertise
- Requires proper indexing strategy
- Need connection pooling (HikariCP)

---

### ADR-006: JDK 21 with Virtual Threads

**Decision:** Use JDK 21 and enable virtual threads.

**Rationale:**
- **Concurrency:** Handle 10,000+ concurrent requests
- **Simplicity:** Write blocking code that performs like async
- **Resource Efficiency:** Lower memory footprint
- **Future-Proof:** Latest LTS with long support

**Consequences:**
- Requires JDK 21+ in production
- Some libraries may not be fully compatible yet
- Need monitoring for virtual thread performance

---

## Conclusion

This backend architecture provides FitHub with:

✅ **Scalability:** Event-driven architecture + CQRS enables horizontal scaling
✅ **Maintainability:** Clean separation of concerns through hexagonal architecture
✅ **Testability:** Domain logic completely independent of infrastructure
✅ **Flexibility:** Easy to swap implementations without changing business logic
✅ **Cultural Compliance:** Arabic language, Hijri calendar, prayer time support
✅ **Regulatory Compliance:** Multi-tenant isolation, audit trail, ZATCA integration
✅ **Developer Experience:** Modern Kotlin with Spring Boot 3.5.6 and JDK 21

The architecture is designed to evolve with FitHub's growth from 10 to 1000+ facilities while maintaining code quality, performance, and reliability.
