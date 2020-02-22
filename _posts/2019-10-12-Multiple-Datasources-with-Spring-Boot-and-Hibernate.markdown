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