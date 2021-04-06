## Feign

# 1. Visão Geral
Neste tutorial, apresentaremos o Feign - um cliente HTTP declarativo desenvolvido pela Netflix.

O Feign visa simplificar os clientes de API HTTP. Simplificando, o desenvolvedor precisa apenas declarar e anotar uma interface enquanto a implementação real é provisionada no tempo de execução.

# 2. Exemplo
Ao longo deste tutorial, usaremos um aplicativo de livraria de exemplo que expõe o ponto de extremidade da API REST.

Podemos clonar facilmente o projeto e executá-lo localmente:

```
mvn install spring-boot:run
```

# 3. Configuração
Primeiro, vamos adicionar as dependências necessárias:

```
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-okhttp</artifactId>
    <version>10.11</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-gson</artifactId>
    <version>10.11</version>
</dependency>
<dependency>
    <groupId>io.github.openfeign</groupId>
    <artifactId>feign-slf4j</artifactId>
    <version>10.11</version>
</dependency>
```

Além da dependência feign-core (que também é puxada), usaremos alguns plug-ins, especialmente: feign-okhttp para usar internamente o cliente OkHttp do Square para fazer solicitações, feign-gson para usar GSON do Google como processador JSON e feign- slf4j por usar a Fachada de Log Simples para registrar solicitações.

Para realmente obter alguma saída de log, precisaremos de nossa implementação de logger suportada por SLF4J favorita no caminho de classe.

Antes de prosseguirmos com a criação de nossa interface de cliente, primeiro vamos configurar um modelo de livro para armazenar os dados:

```
public class Book {
    private String isbn;
    private String author;
    private String title;
    private String synopsis;
    private String language;

    // standard constructor, getters and setters
}
```

### NOTA
Pelo menos um “construtor sem argumentos” é necessário para um processador JSON.


Na verdade, nosso provedor REST é uma API orientada a hipermídia, portanto, precisaremos adicionalmente de uma classe de wrapper simples:

```
public class BookResource {
    private Book book;

    // standard constructor, getters and setters
}
```

### Nota
Manteremos o BookResource simples porque nosso cliente Feign de amostra não se beneficia dos recursos de hipermídia!

# 4. Lado do servidor
Para entender como definir um cliente Feign, primeiro veremos alguns dos métodos e respostas suportados por nosso provedor REST.

Vamos experimentar com um simples comando curl shell para listar todos os livros. Precisamos lembrar de prefixar todas as chamadas com /api, que é o contexto de servlet do aplicativo:

```
curl http://localhost:8081/api/books
```

Como resultado, obteremos um repositório de livros completo representado como JSON:

```
[
  {
    "book": {
      "isbn": "1447264533",
      "author": "Margaret Mitchell",
      "title": "Gone with the Wind",
      "synopsis": null,
      "language": null
    },
    "links": [
      {
        "rel": "self",
        "href": "http://localhost:8081/api/books/1447264533"
      }
    ]
  },

  ...

  {
    "book": {
      "isbn": "0451524934",
      "author": "George Orwell",
      "title": "1984",
      "synopsis": null,
      "language": null
    },
    "links": [
      {
        "rel": "self",
        "href": "http://localhost:8081/api/books/0451524934"
      }
    ]
  }
]
```

Também podemos consultar recursos de livros individuais, anexando o ISBN a uma solicitação get:

```
curl http://localhost:8081/api/books/1447264533
```

# 5. Fingir Cliente
Finalmente, vamos definir nosso cliente Feign.

Usaremos a anotação @RequestLine para especificar o verbo HTTP e uma parte do caminho como argumento. Os parâmetros serão modelados usando a anotação @Param:

```
public interface BookClient {
    @RequestLine("GET /{isbn}")
    BookResource findByIsbn(@Param("isbn") String isbn);

    @RequestLine("GET")
    List<BookResource> findAll();

    @RequestLine("POST")
    @Headers("Content-Type: application/json")
    void create(Book book);
}
```

### NOTA
Os clientes Feign podem ser usados para consumir APIs HTTP baseadas em texto apenas, o que significa que eles não podem lidar com dados binários, por exemplo, uploads ou downloads de arquivos.


Isso é tudo! Agora usaremos o Feign.builder() para configurar nosso cliente baseado em interface. A implementação real será provisionada no tempo de execução:

```
BookClient bookClient = Feign.builder()
  .client(new OkHttpClient())
  .encoder(new GsonEncoder())
  .decoder(new GsonDecoder())
  .logger(new Slf4jLogger(BookClient.class))
  .logLevel(Logger.Level.FULL)
  .target(BookClient.class, "http://localhost:8081/api/books");
```

Feign oferece suporte a vários plug-ins, como codificadores e decodificadores JSON/XML ou um cliente HTTP subjacente para fazer as solicitações.

# 6. Teste de Unidade
Vamos criar três casos de teste para testar nosso cliente. Observe que usamos importações estáticas para org.hamcrest.CoreMatchers. * E org.junit.Assert. *:

```
@Test
public void givenBookClient_shouldRunSuccessfully() throws Exception {
   List<Book> books = bookClient.findAll().stream()
     .map(BookResource::getBook)
     .collect(Collectors.toList());

   assertTrue(books.size() > 2);
}

@Test
public void givenBookClient_shouldFindOneBook() throws Exception {
    Book book = bookClient.findByIsbn("0151072558").getBook();
    assertThat(book.getAuthor(), containsString("Orwell"));
}

@Test
public void givenBookClient_shouldPostBook() throws Exception {
    String isbn = UUID.randomUUID().toString();
    Book book = new Book(isbn, "Me", "It's me!", null, null);
    bookClient.create(book);
    book = bookClient.findByIsbn(isbn).getBook();

    assertThat(book.getAuthor(), is("Me"));
}
```

# 7. Leitura Adicional
Se precisarmos de algum tipo de fallback no caso de indisponibilidade do serviço, poderíamos adicionar HystrixFeign ao classpath e construir nosso cliente com HystrixFeign.builder().

Além disso, também é possível adicionar balanceamento de carga do lado do cliente e / ou descoberta de serviço ao nosso cliente.

Poderíamos conseguir isso adicionando Ribbon ao nosso classpath e usar o construtor assim:

```
BookClient bookClient = Feign.builder()
  .client(RibbonClient.create())
  .target(BookClient.class, "http://localhost:8081/api/books");
```

Para descoberta de serviço, temos que construir nosso serviço com Spring Cloud Netflix Eureka habilitado. Em seguida, basta integrar com Spring Cloud Netflix Feign. Como resultado, temos o balanceamento de carga da faixa de opções gratuitamente. Mais sobre isso pode ser encontrado aqui.

# 8. Conclusão

Neste artigo, explicamos como construir um cliente HTTP declarativo usando Feign para consumir APIs baseadas em texto.