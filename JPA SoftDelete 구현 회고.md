SpringBoot JPA에서 SoftDelete 구현 시 경험한 내용을 정리한 문서입니다.

### 서론

신규 프로젝트에 투입되어 그동안 많이 사용해보지 않았던 Kotlin과 SpringBoot 그리고 JPA를 사용하여 개발을 진행하고 있던 중 SoftDelete의 구현을 위해 SQLRestriction Annotation을 사용하였습니다.

그 과정에서 경험한 내용을 정리하고자 합니다.

### SQLRestriction 사용

SQLRestriction Annotation은 Hibernate에서 제공하는 Annotation으로, Entity 수준에서 SQL 조건을 고정적으로 적용하는 Annotation입니다.

Entity에 대한 모든 쿼리에 SQL 조건을 자동으로 적용하므로, SoftDelete와 같은 기능을 구현할 때 유용하게 사용할 수 있습니다.

다음은 SQLRestriction Annotation의 동작 순서입니다.

1. Hibernate에서 런타임 시점에 Entity 클래스의 Annotation을 읽어들입니다.
2. 이 과정에서 SQLRestriction Annotation을 읽어들이고, 해당 Entity의 메타데이터에 지정된 조건을 저장합니다.
3. 이후 해당 Entity에 대한 쿼리를 실행할 때, 저장된 조건을 적용하여 쿼리를 실행합니다.

다음은 SQLRestriction Annotation을 사용하여 SoftDelete를 구현한 예시입니다.

```kotlin
@SQLDelete(sql = "UPDATE users SET deleted_at = NOW() WHERE id = ?")
@SQLRestriction(value = "\"deleted_at\" IS NULL")
class User(
    ...

    @Column(nullable = true)
    var deletedAt : OffsetDateTime? = null
)
```

### SQLRestriction의 한계

이처럼 SQLRestriction Annotation을 사용하여 간단하게 조건을 적용할 수 있지만, SQLRestriction Annotation의 설계 의도에 따라 특정 쿼리에 대해서는 조건을 제외해야 하는 경우 문제가 발생할 수 있습니다.

예를 들어 백오피스에서 삭제된 데이터를 조회해야 하는 경우 NativeQuery를 제외한 방법으로는 SQLRestriction Annotation에 의해 조회가 불가능합니다.

이는 SQLRestriction Annotation의 설계 의도가 Entity 수준에서 고정적인 조건을 적용하는 것이기 때문에 발생하는 것으로 간편하게 고정된 조건을 적용하여 성능 및 안정성을 확보하는 것이 목적입니다.

### 대안

SQLRestriction Annotation의 한계로 인해 삭제된 데이터를 조회해야 하는 경우 다음과 같은 대안을 사용할 수 있습니다.

1. NativeQuery를 사용하여 직접 쿼리를 작성하는 방법
2. 백오피스 전용 WithDeleted Entity를 생성하여 조회하는 방법
3. Specification을 사용하여 동적으로 조건을 적용하는 방법
4. Filter Annotation을 사용하여 동적으로 조건을 적용하는 방법

### 대안 1. nativeQuery 사용

nativeQuery를 사용하여 직접 쿼리를 작성하는 방법은 가장 간단하게 해결할 수 있는 방법입니다.

다음은 nativeQuery를 사용하여 DeletedAt이 NOT NULL인 User를 조회하는 예시입니다.

```kotlin
// UserRepository.kt
@Repository
interface UserRepository : JpaRepository<User, Long> {
    @Query(value = "SELECT * FROM users WHERE deleted_at IS NOT NULL", nativeQuery = true)
    fun findAllDeleted() : List<User>
}
```

JPQL을 통해 직접 쿼리를 작성한 뒤 nativeQuery를 true로 설정하여 쿼리를 실행하면, SQLRestriction Annotation에 의해 제한된 쿼리를 실행할 수 있습니다.

하지만 이 방법은 다수의 Entity에 대해 삭제된 데이터를 조회해야 하는 경우 해당하는 모든 Repository에 쿼리를 직접 작성해야 하므로 유지보수성이 떨어지며

JPA의 장점인 타입에 안전한 쿼리를 사용할 수 없는 단점이 있습니다.

또한 백오피스의 특성상 페이징 처리가 필요한 경우 Specification 및 Pageable을 사용할 수 없다는 단점이 있습니다.

### 대안 2. WithDeleted Entity 생성

SQLRestriction Annotation이 적용되지 않은 Entity를 생성하고 Repository를 분리하여 조회하는 방법은 SQLRestriction Annotation의 한계를 극복할 수 있는 방법 중 하나입니다.

다음은 WithDeleted Entity를 생성하여 조회하는 예시입니다.

```kotlin
// WithDeletedUser.kt
@Entity
@Table(name = "users")
class WithDeletedUser(
    ...

    @Column(nullable = true)
    var deletedAt : OffsetDateTime? = null
)

// WithDeletedUserRepository.kt
@Repository
interface WithDeletedUserRepository : JpaRepository<WithDeletedUser, Long> {
    fun findAllByDeletedAtIsNotNull() : List<WithDeletedUser>
}
```

SQLRestriction Annotation이 적용되지 않은 Entity를 사용하여 조회하는 방법은 nativeQuery 사용 시 발생하는 대부분의 단점을 극복할 수 있지만

삭제된 데이터의 조회가 필요한 Entity마다 WithDeleted Entity를 생성해야 하므로 중복된 Entity를 다수 생성해야 하는 단점이 있습니다.

### 대안 3. Specification 사용

Specification을 사용하여 동적으로 조건을 적용하는 방법은 JPA의 Criteria API를 사용하여 동적으로 쿼리를 생성하는 방법입니다.

다음은 Specification을 사용하여 DeletedAt이 포함된 User를 조회하는 예시입니다.

```kotlin
// UserSpecification.kt
val includeDeletedSpec = Specification<User> { root, _, criteriaBuilder ->
    criteriaBuilder.conjunction()  // 모든 데이터를 조회하도록 조건 없음
}

val excludeDeletedSpec = Specification<User> { root, _, criteriaBuilder ->
    criteriaBuilder.isNull(root.get<LocalDateTime>("deletedAt"))
}

// UserRepository.kt
@Repository
interface UserRepository : JpaRepository<User, Long>, JpaSpecificationExecutor<User> {
    fun findAll(spec: Specification<User>) : List<User>
}

// UserService.kt
@Service
class UserService {
    ...

    fun findAllUsers(includeDeleted: Boolean) : List<User> {
        val spec = if (includeDeleted) includeDeletedSpec else excludeDeletedSpec
        return userRepository.findAll(spec)
    }
}
```

이 경우 User Entity에 SQLRestriction Annotation을 적용하지 않고, Specification을 사용하여 동적으로 조건을 적용할 수 있습니다.

하지만 Repository 메서드를 호출할 때마다 Specification을 적용해야하며 Repository 메서드마다 Specification을 생성해야 하므로 코드가 복잡해질 수 있습니다.

### 대안 4. Filter Annotation 사용

Hibernate에서 제공하는 Filter Annotation을 사용하여 동적으로 조건을 적용하는 방법은 SQLRestriction Annotation의 한계를 극복할 수 있는 방법 중 하나입니다.

Filter Annotation은 Entity 수준이 아닌 메서드 수준에서 적용할 수 있어 동적으로 조건을 적용할 수 있습니다.

다음은 서비스 메서드에서 Filter Annotation을 사용하여 DeletedAt을 기본적으로 제외하는 예시입니다.

```kotlin
// User.kt
@Entity
@Table(name = "users")
@FilterDef(name = "withOutDeleted")
@Filter(name = "withOutDeleted", condition = "deleted_at IS NULL")
class User(
    ...

    @Column(nullable = true)
    var deletedAt : OffsetDateTime? = null
)

// UserService.kt
@Service
class UserService(
    val userRepository: UserRepository,
    val entityManager: EntityManager
) {
    ...

    fun findWithOutDeletedUsers() : List<User> {
        val session = entityManager.unwrap(Session::class.java)
        val filter = session.enableFilter("withOutDeleted")
        return userRepository.findAll()
    }
}
```

서비스 수준에서 Filter Annotation을 적용할 경우 코드의 복잡도가 증가할 수 있으나 AOP등을 사용하여 중복을 제거할 수 있습니다.

또한 Repository 메서드를 호출할 때마다 Filter Annotation을 적용할 필요가 없으므로 Specification을 사용하는 방법보다 코드의 복잡도가 낮아질 수 있습니다.

### 결론

SQLRestriction Annotation을 사용하여 SoftDelete를 구현할 경우, SQLRestriction Annotation의 한계로 인해 삭제된 데이터를 조회해야 하는 경우 다양한 대안을 사용할 수 있습니다.

각 대안마다 장단점이 존재하지만 특정 상황에 따라 적절한 대안을 선택하여 사용하면 됩니다.

저는 Filter Annotation을 사용하여 동적으로 조건을 적용하는 방법을 사용하였으며, AOP를 통해 백오피스를 제외한 서비스에 대해 Filter Annotation을 적용하였습니다.

다음 회고에서는 Filter Annotation의 사용과 AOP를 통한 중복 제거 그리고 Filter Annotation의 한계에 대해 정리하고자 합니다.