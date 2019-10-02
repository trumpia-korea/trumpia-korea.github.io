---
title: "Multitenancy in hibernate"
categories:
  - knowledge
tags:
  - hibernate
  - multitenancy
  - java
author: seolmin
---



multitenancy를 직역하자면 '다중 입주' 정도가 될 것 같습니다. 소프트웨어 용어에서는 여러 사용자 그룹이 하나의 소프트웨어를 사용하더라도 각 사용자 그룹을 격리시킬 수 있는 아키텍쳐를 의미합니다. tenant는 각 사용자 그룹을 뜻합니다. hibernate multitenancy는 여러 사용자 그룹이 하나의 서비스용 어플리케이션을 사용하되 각 사용자 그룹별로 사용되는 데이터베이스 데이터는 격리시키는 방법을 제공합니다.



서비스를 운영하다 보면 같은 기능을 제공하지만 고객의 지역이나 중요도와 같은 특성에 따라 다수의 데이터베이스를 나누어 사용하게 됩니다.  Trumpia에서도 고객 그룹에 따라 여러 데이터베이스를 운영하고 있습니다. 기존에는 접근하고자 하는 데이터베이스를 비즈니스 로직에서 제어를 하고 있었고 이는 접근 하고자 하는 데이터베이스 종류들에 의존성을 가지며 각 데이터베이스가 고객 그룹별로 완전히 격리되지 않았습니다. 여기에 hibernate multitenancy를 적용하여 접근 방식을 수정하였고 데이터에 접근해야 하는 비즈니스 로직은 구현해야할 요구사항에만 집중할 수 있게 되었습니다.



이 글에서는 hibernate multitenancy의 개념과 적용 방법을 간단하게 알아보도록 하겠습니다.  여기에는 hibernate5 이상의 버전이 필요합니다.



## Three approaches to multitenancy

hibernate에서는 총 세가지의 multitenancy 접근 방법을 제공합니다. 각기 접근 방법을 살펴보면서 개념을 좀 더 살펴보겠습니다.



### Separate database

tenant별로 물리적으로 나뉘어져 있는 데이터베이스 인스턴스를 사용하는 경우에 해당합니다. 각 tenant는 자신의 데이터베이스 인스턴스에만 접근하도록 합니다. 데이터 접근 계층은 multitenancy에 대해 신경쓰지 않아도 됩니다.

![separate_database](/images/2019-10-02-multitenancy_in_hibernate/separate_database.png)



### Separate schema

단일 데이터베이스 인스턴스에서 tenant별로 각기 다른 스키마를 사용하는 경우에 해당합니다. 각 tenant는 자신의 스키마에만 접근 할 수 있습니다. Separate database와 마찬가지로 데이터 접근 계층은 multitenancy에 신경쓰지 않아도 됩니다.

![separate_schema](/images/2019-10-02-multitenancy_in_hibernate/separate_schema.png)



### Partitioned (discriminator) data

각 tenant는 단일 데이터베이스, 단일 스키마를 사용하되 tenant를 식별하기 위해 테이블의 열이나 특정 SQL수식을 이용하는 경우에 해당합니다. 때문에 공유되는 모든 테이블에 식별을 위한 열이 추가되어야 하고 테이블이나 인덱스의 크기가 커질 수 있습니다.

![partitioned_data](/images/2019-10-02-multitenancy_in_hibernate/partitioned_data.png)



## Implement multitenancy 

multitenancy를 구현하기 위해서는 hibernate에서 제공하는 두가지 interface 구현체를 제공하면 됩니다. 여기서는 Separate database 접근 방식과 스프링 부트를 통한 예제를 살펴보도록 하겠습니다.



### MultiTenantConnectionProvider

`MultiTenantConnectionProvider`는 tenant별 connection을 제공하는 역할을 합니다. `getConnection(String tenantIdentifier)` 호출 시 `tenantIdentifier`에 맞는 connection을 반환하는 메서드를 구현하게 되어 있습니다.  각 dataSource를 map 같은 자료구조에 넣어두었다가 `tenantIdentifier` 에 따라 connection을 반환하도록 구현하면 됩니다.

```java
public interface MultiTenantConnectionProvider extends Service, Wrapped {
    
	public Connection getConnection(String tenantIdentifier) throws SQLException;
    
    ...
}
```



### CurrentTenantIdentifierResolver

tenant 식별자를 제공하는 역할을 합니다. `resolveCurrentTenantIdentifier()` 호출 시 tenant 식별자를 반환하도록 메서드를 구현하게 되어 있습니다. 

```java
public interface CurrentTenantIdentifierResolver {
    
	public String resolveCurrentTenantIdentifier();
    
    ...
}
```

`resolveCurrentTenantIdentifier()`구현을 위해 다음과 같이 `ThreadLocal`을 이용하여 context를 제공하는 방법이 있습니다. 특정 tenant의 태스크 수행 전 `setTenant(String identifier)` 를 호출하여 `TENANT_IDENTIFIER` 를 설정하고 태스크 수행 후 `reset()` 을 호출하여 해제하도록 하는 것입니다.

```java
public class TenantContext {

    private static final ThreadLocal<String> TENANT_IDENTIFIER = new ThreadLocal<>();

    public static void setTenant(String identifier) {

        TENANT_IDENTIFIER.set(identifier);
    }

    public static String getTenant() {

        return Objects.isNull(TENANT_IDENTIFIER.get()) ? DataBaseTenant.DEFAULT : TENANT_IDENTIFIER.get();
    }

    public static void reset() {

        TENANT_IDENTIFIER.remove();
    }
}
```

이를 이용해 `CurrentTenantIdentifierResolver`를 구현하면 다음과 같습니다.

```java
@Component
public class TrumpiaTenantIdentifierResolver implements CurrentTenantIdentifierResolver {

    @Override
    public String resolveCurrentTenantIdentifier() {
        return TenantContext.getTenant();
    }
    
    ...
```



### Properties

multitenancy를 활성화 하기 위해서는 몇가지 hibernate properties 설정이 필요한데 앞서 이야기 했던 접근 방법과 `MultiTenantConnectionProvider`, `CurrentTenantIdentifierResolver` 구현체를 hibernate properties에 넘겨주는 것입니다. 

```properties
hibernate.multiTenancy=DATABASE
hibernate.multi_tenant_connection_provider=foo.bar.TrumpiaMultiTenantConnectionProvider
hibernate.tenant_identifier_resolver=foo.bar.TrumpiaTenantIdentifierResolver
```

다음과 같이 설정할 수도 있습니다.

```java
@Bean
public LocalContainerEntityManagerFactoryBean entityManagerFactory(DataSource dataSource, MultiTenantConnectionProvider tenantConnectionProvider, CurrentTenantIdentifierResolver identifierResolver) {

        Map<String, Object> properties = new HashMap<>(jpaProperties.getProperties());
        properties.put(Environment.MULTI_TENANT, MultiTenancyStrategy.DATABASE);
        properties.put(Environment.MULTI_TENANT_CONNECTION_PROVIDER, tenantConnectionProvider);
        properties.put(Environment.MULTI_TENANT_IDENTIFIER_RESOLVER, identifierResolver);
    
        LocalContainerEntityManagerFactoryBean entityManagerFactoryBean = new LocalContainerEntityManagerFactoryBean();
        entityManagerFactoryBean.setJpaPropertyMap(properties);
    
        ...
```



## Conclusion

hibernate multitenancy의 개념과 구현 예제를 간략하게 살펴보았습니다.  hibernate multitenancy에 대해  자세히 설명된 한글 문서가 없음이 아쉬워 직접 적용해보며 알게 된 정보를 정리해 보았습니다.  

각 고객 그룹별 모두 동일한 기능을 제공하면서 격리된 데이터베이스를 접근할 수 있는 어플리케이션을 운영할 수 있어 매우 유용하다고 생각합니다. 

읽어주셔서 감사하며 Trumpia 기술 블로그를 방문에 주신 분께 도움이 되었길 바랍니다. 아자아자 화이팅!
