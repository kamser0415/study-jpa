# 김영한님 코드
```Java
public class Querydsl4RepositorySupport {
    private final Class domainClass;
    private Querydsl querydsl;
    private EntityManager entityManager;
    private JPAQueryFactory factory;

    public Querydsl4RepositorySupport(Class<?> domainClass) {
        this.domainClass = Assert.notNull(domainClass, "Domain class must not be null!");
    }

    @Autowired
    public void setEntityManager(EntityManager entityManager) {
        Assert.notNull(entityManager, "EntityManager must not be null!");
        JpaEntityInformation entityInformation = JpaEntityInformationSupport.getEntityInformation(domainClass, entityManager);
        SimpleEntityPathResolver resolver = SimpleEntityPathResolver.INSTANCE;
        EntityPath path = resolver.createPath(entityInformation.getJavaType());
        this.entityManager = entityManager;
        this.factory = new JPAQueryFactory(JPQLTemplates.DEFAULT,entityManager);
        this.querydsl = new Querydsl(entityManager, new PathBuilder<Object>(path.getType(), path.getMetadata()));
    }

    @PostConstruct
    public void validate() {
        Assert.notNull(entityManager, "EntityManager must not be null!");
        Assert.notNull(querydsl, "Querydsl must not be null!");
        Assert.notNull(factory, "JPAQueryFactory must not be null!");
    }
    protected JPAQueryFactory getFactory() {
        return factory;
    }
    protected Querydsl getQuerydsl() {
        return querydsl;
    }
    protected EntityManager getEntityManager() {
        return entityManager;
    }
    protected <T> com.querydsl.jpa.JPQLQuery<T> select(com.querydsl.core.types.Expression<T> expr) {
        return getFactory().select(expr);
    }
    protected <T> com.querydsl.jpa.JPQLQuery<T> selectFrom(EntityPath<T> from) {
        return getFactory().selectFrom(from);
    }
    protected <T> Page<T> applyPagination(Pageable pageable, Function<JPAQueryFactory, JPAQuery> contentQuery) {
        JPAQuery query = contentQuery.apply(getFactory());
        List<T> content = getQuerydsl().applyPagination(pageable, query).fetch();
        return PageableExecutionUtils.getPage(content, pageable, query::fetchCount);
    }
    protected <T> Page<T> applyPagination(Pageable pageable, Function<JPAQueryFactory, JPAQuery> contentQuery, Function<JPAQueryFactory, JPAQuery> countQuery) {
        JPAQuery query = contentQuery.apply(getFactory());
        List<T> content = getQuerydsl().applyPagination(pageable, query).fetch();
        return PageableExecutionUtils.getPage(content, pageable, countQuery.apply(getFactory())::fetchCount);
    }
}
```  
