<!--
{
  "title": "Implementing a full text search using Spring Boot 2 + Hibernate Search",
  "subtitle": "A full text search analyzer customization using Java 11, Spring Boot 2 and Hibernate Search",
  "date": "2020-09-13",
  "tags": ["java11", "spring-boot", "hibernate-search", "apache-lucene"],
  "summary": "I needed to implement a full text search mechanism (mainly to be used for auto-complete on clint side) using Java 11 and Spring Boot 2. But I did not find a proper working example with these tech stack and wanted to describe it here."
}
-->

I needed to implement a full text search mechanism (mainly to be used for auto-complete on clint side) using Java 11 and
Spring Boot 2. But I did not find a proper working example with these tech stack and wanted to describe it here.

The requirement:
- case-insensitive search
- search only the connected keywords (i.e. the keyword `new here` should not be found in `new content here` 
but for example `content he` should be matched)

## Background

Hibernate Search uses `Apache Lucene` (as well as `Apache Solr`) to create indexes and build the query to search. 
[Tokenizers](https://lucene.apache.org/solr/guide/6_6/tokenizers.html) are used to process the field or the search keyword so that a desired search result can be achieved
[Filters](https://lucene.apache.org/solr/guide/6_6/filter-descriptions.html) are used to make operations on the tokenized fields.

## Implementation
We will implement a simple full text search service on `Comment` entity.
The full code can be found in this [repository](https://github.com/tahsinozden/spring-boot-hibernate-search).

### Creating a config file
hibernate.properties
```text
hibernate.search.default.directory_provider=filesystem
hibernate.search.default.indexBase=./indexes
```

### Creating a custom analyzer
`KeywordTokenizerFactory` will treat the word as whole (the standard analyzers parse and splits the words)
To achieve the case-insensitive search `LowerCaseFilterFactory` is required. `NGramFilterFactory` will generate ngram
indexes. 
> `minGramSize` and `maxGramSize` define the length of the Ngrams, meaning that full text search will not find any words 
> outside of this interval `minGramSize <= wordLength < maxGramSize`
```java
@Indexed
@AnalyzerDef(name = "customNgramIndexer",
        // consider searching the full text
        tokenizer = @TokenizerDef(factory = KeywordTokenizerFactory.class),
        filters = {
                // make all lower case
                @TokenFilterDef(factory = LowerCaseFilterFactory.class),
                // generate ngram indexes
                @TokenFilterDef(factory = NGramFilterFactory.class,
                        params = {
                                @Parameter(name = "minGramSize", value = "3"),
                                @Parameter(name = "maxGramSize", value = "40")})
        })
@AnalyzerDef(name = "customNgramQuery",
        // consider searching the full text
        tokenizer = @TokenizerDef(factory = KeywordTokenizerFactory.class),
        filters = {
                // make all lower case
                @TokenFilterDef(factory = LowerCaseFilterFactory.class)
        })
// currently Hibernate searches analyzers only where entities are defined. That's why I created an entity here instead of
// defining it inside Comment class
@Entity
@Table(name = "analyzer_definition")
public class AnalyzerDefinition {
    @Id
    private Long id;
}

```

### Creating a full text service
The gotcha here is to use different analyzer definitions to create indexes and query the words. (the difference is that
we should not use Ngram filter while searching, the rest is the same)

```java
@Service
public class CommentFullTextSearchService {

    private final EntityManager entityManager;
    private final FullTextEntityManager fullTextEntityManager;

    @Autowired
    public CommentFullTextSearchService(EntityManagerFactory entityManagerFactory) {
        this.entityManager = entityManagerFactory.createEntityManager();
        this.fullTextEntityManager = Search.getFullTextEntityManager(entityManager);
    }

    public void createOrUpdate() throws InterruptedException {
        fullTextEntityManager.createIndexer().startAndWait();
    }

    public List<Comment> search(String keyword, List<String> fields) {
        EntityContext entityContext = fullTextEntityManager.getSearchFactory()
                .buildQueryBuilder()
                .forEntity(Comment.class);

        // for Ngram filters, the query must not include that filter, otherwise it may fetch all the records relevant or
        // not. this is how to override the analyzer for the field
        fields.forEach(field -> entityContext.overridesForField(field, "customNgramQuery"));
        QueryBuilder queryBuilder = entityContext.get();

        Query query = queryBuilder
                .keyword()
                .onFields(fields.toArray(String[]::new))
                .matching(keyword)
                .createQuery();

        FullTextQuery jpaQuery
                = fullTextEntityManager.createFullTextQuery(query, Comment.class);

        return jpaQuery.getResultList();
    }
}
```

### Using the custom analyzer in the entity
`@Indexed` and `@Field(analyzer = @Analyzer(definition = "customNgramIndexer"))` are required to make the entity searchable

```java
@Indexed
@Entity
@Table(name = "comments")
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Comment {

    @Id
    @GeneratedValue
    private Integer id;

    @Column
    @Field(analyzer = @Analyzer(definition = "customNgramIndexer"))
    private String title;

    @Column
    @Field(analyzer = @Analyzer(definition = "customNgramIndexer"))
    private String author;

    @Column
    @Field(analyzer = @Analyzer(definition = "customNgramIndexer"))
    private String content;
}
```

### Testing
Thanks to the customized analyzer, we can search the words case-insensitive and partially.

```java
@ExtendWith(SpringExtension.class)
@SpringBootTest
class CommentFullTextSearchServiceTest {

    @Autowired
    private CommentRepository commentRepository;

    @Autowired
    private CommentFullTextSearchService commentFullTextSearchService;

    @BeforeEach
    public void beforeEachTest() throws InterruptedException {
        commentRepository.deleteAll();
        List<Comment> comments = List.of(
                new Comment(null, "Wrong item", "Jack", "here is a wrong item"),
                new Comment(null, "good value / price", "a smart developer", "really good content"),
                new Comment(null, "recommends the seller", "John Doe here", "public comment"),
                new Comment(null, "not happy with the product", "Cesar", "definitely you need to find sth better"),
                new Comment(null, "happy with the product here", "Cesar", "find something else")
        );
        commentRepository.saveAll(comments);
        commentFullTextSearchService.createOrUpdate();
    }

    @Test
    public void shouldSearchTitleInTheComment() {
        // given
        String keyword = "c com";

        // when
        List<Comment> actual = commentFullTextSearchService.search(keyword, List.of("content"));

        // then
        assertThat(actual).hasSize(1);
        assertThat(actual.get(0).getContent()).isEqualTo("public comment");
    }

    @Test
    public void shouldSearchAuthorInTheComment() {
        // given
        String keyword = "ces";

        // when
        List<Comment> actual = commentFullTextSearchService.search(keyword, List.of("author"));

        // then
        assertThat(actual).hasSize(2);
        actual.forEach(comment -> assertThat(comment.getAuthor()).isEqualTo("Cesar"));
    }

    @Test
    public void shouldSearchKeywordInSeveralFields() {
        // given
        String keyword = "happy ";

        // when
        List<Comment> actual = commentFullTextSearchService.search(keyword, List.of("title", "author", "content"));

        // then
        assertThat(actual).hasSize(2);
    }

    @Test
    public void shouldSearchKeywordCaseInsensitiveInSeveralFields() {
        // given
        String keyword = "HerE";

        // when
        List<Comment> actual = commentFullTextSearchService.search(keyword, List.of("title", "author", "content"));

        // then
        assertThat(actual).hasSize(3);
    }
}
```






