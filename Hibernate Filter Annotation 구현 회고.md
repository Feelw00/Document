본 문서는 Hibernate의 Filter Annotation 구현과 구현 시 발생한 이슈에 대해 정리한 문서입니다.

### 서론

지난 문서에서는 SpringBoot JPA에서 SoftDelete 구현을 위해 Hibernate의 Filter Annotation을 사용하는 것으로 결정하였습니다.

이번 문서에서는 Hibernate의 Filter Annotation을 사용하여 SoftDelete를 구현하는 과정과 그 과정에서 발생한 이슈에 대해 정리하고자 합니다.

### Filter 사용

Hibernate의 Filter Annotation은 Entity 수준에서 동적으로 조건을 적용할 수 있는 Annotation입니다.

다음은 Filter Annotation을 사용하여 SoftDelete를 구현한 예시입니다.

```kotlin
@FilterDef(name = "withOutDeleted")
@Filter(name = "withOutDeleted", condition = "deleted_at IS NULL")
@Entity
@Table(name = "users")
class User(
    ...
    
    @Column(nullable = true)
    var deletedAt : OffsetDateTime? = null
)
```

위 코드에서 `@FilterDef` Annotation은 Filter의 이름과 파라미터를 정의하고, `@Filter` Annotation은 Filter의 조건을 정의합니다.

저는 단순히 삭제되지 않은 데이터 조회만 필요하므로 파라미터를 사용하지 않고 `deleted_at IS NULL` 조건을 사용하였습니다.

### Filter Name 중복 이슈

Hibernate의 Filter Annotation은 다른 Entity라고 하더라도 동일한 이름의 Filter를 사용할 수 없습니다.

따라서 각각의 Entity마다 다른 이름의 Filter를 사용해야하며 동일한 기능의 Filter를 사용할 경우 Filter가 적용된 Entity를 별도록 생성하고 상속하여 사용하는 것이 좋습니다.

```kotlin
// BaseSoftDeletableEntity.kt
@MappedSuperclass
@EntityListeners(AuditingEntityListener::class)
@FilterDef(name = "withOutDeleted")
@Filter(name = "withOutDeleted", condition = "deleted_at IS NULL")
abstract class BaseSoftDeletableEntity {
    @Column(nullable = true)
    var deletedAt : OffsetDateTime? = null
}
```

```kotlin
// User.kt
@Entity
@Table(name = "users")
class User(
    ...
) : BaseSoftDeletableEntity()
```

이 경우 BaseSoftDeletableEntity를 상속받은 Entity는 `withOutDeleted` Filter를 사용할 수 있습니다.

### 메서드 단위의 Filter 적용

Hibernate의 Filter Annotation은 Entity 수준이 아닌 메서드 수준에서 적용할 수 있습니다.

따라서 Controller나 Service에서 Filter를 적용할 수 있으며 상황에 따라 유동적으로 Filter를 적용할 수 있습니다.

다음은 서비스 수준에서 Filter를 적용하는 예시입니다.

```kotlin
@Service
class UserService (
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

서비스 메서드에서 `EntityManager`를 사용하여 `Session`을 얻어 `enableFilter` 메서드를 사용하여 Filter를 적용할 수 있습니다.

### AOP를 통한 Filter 적용

서비스 메서드에서 Filter를 적용할 경우 코드의 복잡도가 증가할 수 있습니다.

따라서 AOP를 사용하여 패키지나 클래스, 또는 Custom Annotation을 통해 Filter를 적용할 수 있습니다.

다음은 AOP를 사용하여 api 모듈의 모든 서비스 메서드에 Filter를 적용하는 예시입니다.

```kotlin
// HibernateFilterAspect.kt
@Aspect
@Component
class HibernateFilterAspect(
    private val entityManager: EntityManager
) {
    @Before("execution(* com.example.demo.api.services.*(..))")
    fun enableFilter() {
        val session = entityManager.unwrap(org.hibernate.Session::class.java)
        session.enableFilter("withOutDeleted")
    }

    @After("execution(* com.example.demo.api.services.*(..))")
    fun disableFilter() {
        val session = entityManager.unwrap(org.hibernate.Session::class.java)
        session.disableFilter("withOutDeleted")
    }
}
```

이 방법을 통해 백오피스를 제외한 모든 서비스 API에 대해 Filter를 적용할 수 있습니다.

### Custom Annotation을 통한 Filter 적용

AOP를 사용하여 Filter를 적용할 경우 모든 서비스 메서드에 Filter가 적용되므로 특정 서비스 메서드에만 Filter를 적용하고 싶은 경우 Custom Annotation을 사용할 수 있습니다.

다음은 Custom Annotation을 사용하여 Filter를 적용하는 예시입니다.

```kotlin
// WithOutDeleted.kt
@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class WithOutDeleted
```

```kotlin
// HibernateFilterAspect.kt
@Aspect
@Component
class HibernateFilterAspect(
    private val entityManager: EntityManager
) {

    @Pointcut("@annotation(com.example.demo.api.annotations.WithOutDeleted)")
    fun withOutDeleted() {}

    @Before("withOutDeleted()")
    fun enableFilter() {
        val session = entityManager.unwrap(org.hibernate.Session::class.java)
        session.enableFilter("withOutDeleted")
    }

    @After("withOutDeleted()")
    fun disableFilter() {
        val session = entityManager.unwrap(org.hibernate.Session::class.java)
        session.disableFilter("withOutDeleted")
    }
}
```

```kotlin
@Service
class UserService {
    ...
    
    @WithOutDeleted
    fun findWithOutDeletedUsers() : List<User> {
        return userRepository.findAll()
    }
}
```

이 방법을 통해 특정 서비스 메서드에만 Filter를 적용할 수 있습니다.

다만 저는 백오피스를 제외한 서비스에 대해 Filter를 적용해야 하므로 AOP를 사용하여 모든 서비스 메서드에 Filter를 적용하였습니다.

### JPA findById() 메서드 사용 시 Filter 미적용 이슈

JPA의 `findById()` 메서드를 사용하여 Entity를 조회할 경우 Filter가 적용되지 않는 이슈가 발생하였습니다.

이는 `findById()` 메서드가 `EntityManager`의 `find()` 메서드를 사용하여 Entity를 조회하기 때문에 발생한 이슈로 Hibernate의 Filter가 엔티티를 직접 조회하는 메서드인 `EntityManager.find()` 또는 `Session.get()` 메서드에는 적용되지 않기 때문입니다.

Hibernate의 Filter는 JPQL, HQL, Criteria API를 사용하여 Entity를 조회할 때만 적용됩니다.

이는 Hibernate 개발팀에서 ID 조회에 Filter를 적용해서는 안된다고 논의 후 결정한 내용이며 해당 내용은 [링크](https://discourse.hibernate.org/t/find-by-id-and-filters/7632)에서 확인할 수 있습니다.

### 조치 방안

하지만 Filter가 미적용되는 메서드는 `findById()` 메서드 뿐이므로 `findById()` 메서드를 사용하지 않고 JPQL, HQL, Criteria API를 사용하여 Entity를 조회하면 Filter가 적용됩니다.

예를 들어 다음과 같이 Filter가 적용된 Entity를 조회할 수 있습니다.

```kotlin
@Repository
interface UserRepository : JpaRepository<User, Long> {
    fun findUserById(id: Long) : User?
}
```

이처럼 `findById()` 메서드를 사용하지 않고 직접 메서드를 정의하여 Entity를 조회하면 Filter가 적용됩니다.

아니면 단순히 `findById()` 메서드를 사용하고 DeleteAt을 확인하여 DeletedAt이 null이 아닌 경우 예외를 발생시키거나 null을 반환하도록 할 수 있습니다.

### 마치며

Hibernate의 Filter Annotation을 사용하여 SoftDelete를 구현하는 방법과 그 과정에서 발생한 이슈에 대해 정리하였습니다.

Spring과 Kotlin 환경에서 개발 경험이 길지 않아 찾아보고 시도해보는 과정이 조금 길었지만 재미있는 경험이었습니다.