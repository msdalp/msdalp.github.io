---
layout: post
title:  "Multiple Datasources with Spring Boot and Hibernate"
date:   2019-10-12 12:00:00
categories:
---

Adding two database connection to a Spring Boot application is quite straightforward unless you are using `JpaRepositories`. Simple task of adding urls and giving JdbcTemplate different identifier names become an annoying task.

Starting with adding configurations of two different Datasource to the project: 

{% highlight yaml %}
app.datasource.backup.url=jdbc:mysql://backup_db
app.datasource.backup.username=user
app.datasource.backup.password=password
app.datasource.backup.maximum-pool-size=10

app.datasource.main.url=jdbc:mysql://main_db
app.datasource.main.username=user
app.datasource.main.password=password
app.datasource.main.maximum-pool-size=10
{% endhighlight %}

These will be used for datasource creation initially and later identifiers will be enough. 

{% highlight java %}
    @Bean
    @Primary
    @ConfigurationProperties("app.datasource.main")
    public DataSourceProperties mainDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean(name = "dbMain")
    @Primary
    public DataSource mainDataSource() {
        return mainDataSourceProperties().initializeDataSourceBuilder().build();
    }

    @Bean(name = "jdbcMain")
    @Autowired
    public JdbcTemplate mainJdbcTemplate(@Qualifier("dbMain") DataSource dsMain) {
        return new JdbcTemplate(dsMain);
    }


    @Bean(name = "jdbcBackup")
    @Autowired
    public JdbcTemplate slaveJdbcTemplate(@Qualifier("dbBackup") DataSource dsSlave) {
        return new JdbcTemplate(dsSlave);
    }

    @Bean
    @ConfigurationProperties("app.datasource.backup")
    public DataSourceProperties backupDataSourceProperties() {
        return new DataSourceProperties();
    }


    @Bean(name = "dbBackup")
    public DataSource backupDataSource() {
        return backupDataSourceProperties().initializeDataSourceBuilder().build();
    }
{% endhighlight %}

In a configuration file or in the SpringBoot main class define these datasource beans. 
* Read configuration information for resources
* Create a datasource with a name from that configuration
* Set the datasource created to a JdbcTemplate. The name given here will be used for accessing to the different JdbcTemplate beans as `@Qualifier("jdbcMain") JdbcTemplate jdbcTemplate`

While defining a jdbc repository give the identifier to access a specific database. 

{% highlight java %}
@Repository
public class GameRepository {

    private final JdbcTemplate jdbcTemplate;

    @Autowired
    public GameRepository(@Qualifier("jdbcMain") JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }
    ...
{% endhighlight %}

What if we want to use JPA and define our objects with `Entity`. Datasource definitions and initialization will not change. However different `EntityManagers` and `TransactionManagers` must be defined. 

Annoying part is having auto mapping repositories to these frameworks is done by scanning packages. Which package to scan must be defined clearly otherwise build will fail. Another point is  different entity managers must have different packages as well. It is not possible to use a `com.test.repositories` package and make both scan in there. 

Datasource configuration is almost the same as the previous one but this time let's use configuration file. `DataSourceConfiguration` file will be defined as:

{% highlight java %}
@Configuration
public class DataSourceConfig {

    // first database
    @Primary
    @Bean(name = "blog")
    @ConfigurationProperties("app.datasource.blog")
    public DataSourceProperties firstDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Primary
    @Bean(name = "dataSourceBlogDb")
    public HikariDataSource firstDataSource(@Qualifier("blog") DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }

    // second database
    @Bean(name = "log")
    @ConfigurationProperties("app.datasource.log")
    public DataSourceProperties secondDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean(name = "dataSourceLogDb")
    public HikariDataSource secondDataSource(@Qualifier("log") DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().type(HikariDataSource.class).build();
    }
}
{% endhighlight %}

Next step is defining a EntityManager. Be aware of the base package `io.msdalp.dsone` to be scanned. First entity manager will deal with datasource one and everything related to it will be stored in `io.msdalp.dsone`. Also do not forget to update Hibernate Dialect if you are using a different database. 

{% highlight java %}

@Configuration
@EnableJpaRepositories(
        entityManagerFactoryRef = "ds1EntityManagerFactory",
        transactionManagerRef = "ds1TransactionManager",
        basePackages = "io.msdalp.dsone"
)
@EnableTransactionManagement
public class FirstEntityManagerFactory {

    @Bean
    @Primary
    public LocalContainerEntityManagerFactoryBean ds1EntityManagerFactory(
            EntityManagerFactoryBuilder builder, @Qualifier("dataSourceBlogDb") DataSource dataSource) {

        return builder
                .dataSource(dataSource)
                .packages("io.msdalp.dsone")
                .persistenceUnit("ds1-pu")
                .properties(hibernateProperties())
                .build();
    }

    @Bean
    @Primary
    public PlatformTransactionManager ds1TransactionManager(
            @Qualifier("ds1EntityManagerFactory") EntityManagerFactory ds1EntityManagerFactory) {
        return new JpaTransactionManager(ds1EntityManagerFactory);
    }

    protected Map<String, String> hibernateProperties() {
        return new HashMap<String, String>() {
            {
                put("hibernate.dialect", "org.hibernate.dialect.H2Dialect");
                put("hibernate.hbm2ddl.auto", "create");
            }
        };
    }
}
{% endhighlight %}

Second entity manager will be exactly the same with this one but only different identifiers. It is using the second datasource and scanning `io.msdalp.dstwo`.

{% highlight java %}
@Configuration
@EnableJpaRepositories(
        entityManagerFactoryRef = "ds2EntityManagerFactory",
        transactionManagerRef = "ds2TransactionManager",
        basePackages = "io.msdalp.dstwo"
)
@EnableTransactionManagement
public class SecondEntityManagerFactory {

    @Bean
    public LocalContainerEntityManagerFactoryBean ds2EntityManagerFactory(
            EntityManagerFactoryBuilder builder, @Qualifier("dataSourceLogDb") DataSource dataSource) {

        return builder
                .dataSource(dataSource)
                .packages("io.msdalp.dstwo")
                .persistenceUnit("ds2-pu")
                .properties(hibernateProperties())
                .build();
    }

    @Bean
    public PlatformTransactionManager ds2TransactionManager(
            @Qualifier("ds2EntityManagerFactory") EntityManagerFactory secondEntityManagerFactory) {
        return new JpaTransactionManager(secondEntityManagerFactory);
    }

    protected Map<String, String> hibernateProperties() {
        return new HashMap<String, String>() {
            {
                put("hibernate.dialect", "org.hibernate.dialect.H2Dialect");
                put("hibernate.hbm2ddl.auto", "create");
            }
        };
    }

}
{% endhighlight %}

With these configurations any repository or entity defined under package `io.msdalp.dsone` will work with `Blog` database and `io.msdalp.dstwo` with `Log` database. 

You can check the sample project at [https://github.com/msdalp/spring-multi-datasource](https://github.com/msdalp/spring-multi-datasource).