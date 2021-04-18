## Microsserviços com Oracle Helidon

# 1. Introdução
Helidon é a nova estrutura de microsserviço Java que foi liberada recentemente pela Oracle. Foi usado internamente em projetos Oracle sob o nome J4C (Java for Cloud).

Neste tutorial, cobriremos os principais conceitos da estrutura e, em seguida, passaremos para construir e executar um microsserviço baseado em Helidon.

# 2. Modelo de Programação
Atualmente, o framework oferece suporte a dois modelos de programação para escrever microsserviços: Helidon SE e Helidon MP.

Enquanto Helidon SE é projetado para ser um microframework que suporta o modelo de programação reativa, Helidon MP, por outro lado, é um tempo de execução Eclipse MicroProfile que permite que a comunidade Jakarta EE execute microsserviços de uma forma portátil.

Em ambos os casos, um microsserviço Helidon é um aplicativo Java SE que inicia um servidor HTTP minúsculo a partir do método principal.

# 3. Helidon SE
Nesta seção, descobriremos com mais detalhes os principais componentes do Helidon SE: WebServer, Config e Security.

### 3.1. Configurando o WebServer
Para começar a usar a API WebServer, precisamos adicionar a dependência Maven necessária ao arquivo pom.xml:

```
<dependency>
    <groupId>io.helidon.webserver</groupId>
    <artifactId>helidon-webserver</artifactId>
    <version>0.10.4</version>
</dependency>
```

Para ter uma aplicação web simples, podemos usar um dos seguintes métodos construtores: WebServer.create (serverConfig, routing) ou apenas WebServer.create (routing). O último usa uma configuração de servidor padrão, permitindo que o servidor seja executado em uma porta aleatória.

Aqui está um aplicativo da Web simples que é executado em uma porta predefinida. Também registramos um manipulador simples que responderá com uma mensagem de saudação para qualquer solicitação HTTP com o caminho ‘/ greet’ e Método GET:

```
public static void main(String... args) throws Exception {
    ServerConfiguration serverConfig = ServerConfiguration.builder()
      .port(9001).build();
    Routing routing = Routing.builder()
      .get("/greet", (request, response) -> response.send("Hello World !")).build();
    WebServer.create(serverConfig, routing)
      .start()
      .thenAccept(ws ->
          System.out.println("Server started at: http://localhost:" + ws.port())
      );
}
```

A última linha é iniciar o servidor e aguardar o atendimento das solicitações HTTP. Mas se executarmos este código de amostra no método principal, obteremos o erro:

```
Exception in thread "main" java.lang.IllegalStateException: 
  No implementation found for SPI: io.helidon.webserver.spi.WebServerFactory
```

O WebServer é na verdade um SPI e precisamos fornecer uma implementação de tempo de execução. Atualmente, Helidon fornece a implementação NettyWebServer que é baseada no Netty Core.

Esta é a dependência do Maven para esta implementação:

```
<dependency>
    <groupId>io.helidon.webserver</groupId>
    <artifactId>helidon-webserver-netty</artifactId>
    <version>0.10.4</version>
    <scope>runtime</scope>
</dependency>
```

Agora, podemos executar o aplicativo principal e verificar se ele funciona invocando o endpoint configurado:

```
http://localhost:9001/greet
```

Neste exemplo, configuramos a porta e o caminho usando o padrão do construtor.

O Helidon SE também permite o uso de um padrão de configuração em que os dados de configuração são fornecidos pela API de configuração. Este é o assunto da próxima seção.

### 3.2. A API de configuração
A API Config fornece ferramentas para ler dados de configuração de uma fonte de configuração.

Helidon SE fornece implementações para muitas fontes de configuração. A implementação padrão é fornecida por helidon-config, onde a fonte de configuração é um arquivo application.properties localizado no classpath:

```
<dependency>
    <groupId>io.helidon.config</groupId>
    <artifactId>helidon-config</artifactId>
    <version>0.10.4</version>
</dependency>
```

Para ler os dados de configuração, precisamos apenas usar o construtor padrão que, por padrão, obtém os dados de configuração de application.properties:

```
Config config = Config.builder().build();
```

Vamos criar um arquivo application.properties no diretório src/main/resource com o seguinte conteúdo:

```
server.port=9080
web.debug=true
web.page-size=15
user.home=C:/Users/app
```

Para ler os valores, podemos usar o método Config.get() seguido por uma conversão conveniente para os tipos Java correspondentes:

```
int port = config.get("server.port").asInt();
int pageSize = config.get("web.page-size").asInt();
boolean debug = config.get("web.debug").asBoolean();
String userHome = config.get("user.home").asString();
```

Na verdade, o construtor padrão carrega o primeiro arquivo encontrado nesta ordem de prioridade: application.yaml, application.conf, application.json e application.properties. Os três últimos formatos precisam de uma dependência de configuração extra relacionada. Por exemplo, para usar o formato YAML, precisamos adicionar a dependência de configuração YAML relacionada:

```
<dependency>
    <groupId>io.helidon.config</groupId>
    <artifactId>helidon-config-yaml</artifactId>
    <version>0.10.4</version>
</dependency>
```

E então, adicionamos um application.yml:

```
server:
  port: 9080  
web:
  debug: true
  page-size: 15
user:
  home: C:/Users/app
```

Da mesma forma, para usar o CONF, que é um formato simplificado JSON, ou formatos JSON, precisamos adicionar a dependência helidon-config-hocon.

Observe que os dados de configuração nesses arquivos podem ser substituídos por variáveis de ambiente e propriedades do sistema Java.

Também podemos controlar o comportamento padrão do construtor desativando a variável de ambiente e as propriedades do sistema ou especificando explicitamente a fonte de configuração:

```
ConfigSource configSource = ConfigSources.classpath("application.yaml").build();
Config config = Config.builder()
  .disableSystemPropertiesSource()
  .disableEnvironmentVariablesSource()
  .sources(configSource)
  .build();
```

Além de ler os dados de configuração do classpath, também podemos usar duas configurações de fontes externas, ou seja, as configurações git e etcd. Para isso, precisamos das dependências helidon-config-git e helidon-git-etcd.

Finalmente, se todas essas fontes de configuração não satisfizerem nossa necessidade, o Helidon nos permitirá fornecer uma implementação para nossa fonte de configuração. Por exemplo, podemos fornecer uma implementação que pode ler os dados de configuração de um banco de dados.

### 3.3. A API de roteamento
A API Routing fornece o mecanismo pelo qual vinculamos solicitações HTTP a métodos Java. Podemos fazer isso usando o método de solicitação e o caminho como critérios de correspondência ou o objeto RequestPredicate para usar mais critérios.

Portanto, para configurar uma rota, podemos apenas usar o método HTTP como critério:

```
Routing routing = Routing.builder()
  .get((request, response) -> {} );
```

Ou podemos combinar o método HTTP com o caminho da solicitação:

```
Routing routing = Routing.builder()
  .get("/path", (request, response) -> {} );
```

Também podemos usar o RequestPredicate para obter mais controle. Por exemplo, podemos verificar se há um cabeçalho existente ou o tipo de conteúdo:

```
Routing routing = Routing.builder()
  .post("/save",
    RequestPredicate.whenRequest()
      .containsHeader("header1")
      .containsCookie("cookie1")
      .accepts(MediaType.APPLICATION_JSON)
      .containsQueryParameter("param1")
      .hasContentType("application/json")
      .thenApply((request, response) -> { })
      .otherwise((request, response) -> { }))
      .build();
```

Até agora, fornecemos manipuladores no estilo funcional. Também podemos usar a classe Service, que permite escrever manipuladores de uma maneira mais sofisticada.

Então, vamos primeiro criar um modelo para o objeto com o qual estamos trabalhando, a classe Book:

```
public class Book {
    private String id;
    private String name;
    private String author;
    private Integer pages;
    // ...
}
```

Podemos criar serviços REST para a classe Book implementando o método Service.update(). Isso permite configurar os subcaminhos do mesmo recurso:

```
public class BookResource implements Service {

    private BookManager bookManager = new BookManager();

    @Override
    public void update(Routing.Rules rules) {
        rules
          .get("/", this::books)
          .get("/{id}", this::bookById);
    }

    private void bookById(ServerRequest serverRequest, ServerResponse serverResponse) {
        String id = serverRequest.path().param("id");
        Book book = bookManager.get(id);
        JsonObject jsonObject = from(book);
        serverResponse.send(jsonObject);
    }

    private void books(ServerRequest serverRequest, ServerResponse serverResponse) {
        List<Book> books = bookManager.getAll();
        JsonArray jsonArray = from(books);
        serverResponse.send(jsonArray);
    }
    //...
}
```

Também configuramos o tipo de mídia como JSON, portanto, precisamos da dependência helidon-webserver-json para esta finalidade:

```
<dependency>
    <groupId>io.helidon.webserver</groupId>
    <artifactId>helidon-webserver-json</artifactId>
    <version>0.10.4</version>
</dependency>
```

Finalmente, usamos o método register() do construtor Routing para vincular o caminho raiz ao recurso. Nesse caso, os caminhos configurados pelo serviço são prefixados pelo caminho raiz:

```
Routing routing = Routing.builder()
  .register(JsonSupport.get())
  .register("/books", new BookResource())
  .build();
```

Agora podemos iniciar o servidor e verificar os endpoints:

```
http://localhost:9080/books
http://localhost:9080/books/0001-201810
```

### 3.4. Segurança
Nesta seção, vamos proteger nossos recursos usando o módulo de Segurança.

Vamos começar declarando todas as dependências necessárias:

```
<dependency>
    <groupId>io.helidon.security</groupId>
    <artifactId>helidon-security</artifactId>
    <version>0.10.4</version>
</dependency>
<dependency>
    <groupId>io.helidon.security</groupId>
    <artifactId>helidon-security-provider-http-auth</artifactId>
    <version>0.10.4</version>
</dependency>
<dependency>
    <groupId>io.helidon.security</groupId>
    <artifactId>helidon-security-integration-webserver</artifactId>
    <version>0.10.4</version>
</dependency>
```

As dependências helidon-security, helidon-security-provider-http-auth e helidon-security-integration-webserver estão disponíveis no Maven Central.

O módulo de segurança oferece muitos provedores de autenticação e autorização. Para este exemplo, usaremos o provedor de autenticação básica HTTP, pois é bastante simples, mas o processo para outros provedores é quase o mesmo.

A primeira coisa a fazer é criar uma instância de segurança. Podemos fazer isso de forma programática para simplificar:

```
Map<String, MyUser> users = //...
UserStore store = user -> Optional.ofNullable(users.get(user));

HttpBasicAuthProvider httpBasicAuthProvider = HttpBasicAuthProvider.builder()
  .realm("myRealm")
  .subjectType(SubjectType.USER)
  .userStore(store)
  .build();

Security security = Security.builder()
  .addAuthenticationProvider(httpBasicAuthProvider)
  .build();
```

Ou podemos usar uma abordagem de configuração.

Nesse caso, declararemos todas as configurações de segurança no arquivo application.yml que carregamos por meio da API de configuração:

```
#Config 4 Security ==> Mapped to Security Object
security:
  providers:
  - http-basic-auth:
      realm: "helidon"
      principal-type: USER # Can be USER or SERVICE, default is USER
      users:
      - login: "user"
        password: "user"
        roles: ["ROLE_USER"]
      - login: "admin"
        password: "admin"
        roles: ["ROLE_USER", "ROLE_ADMIN"]

  #Config 4 Security Web Server Integration ==> Mapped to WebSecurity Object
  web-server:
    securityDefaults:
      authenticate: true
    paths:
    - path: "/user"
      methods: ["get"]
      roles-allowed: ["ROLE_USER", "ROLE_ADMIN"]
    - path: "/admin"
      methods: ["get"]
      roles-allowed: ["ROLE_ADMIN"]
```

E para carregá-lo, precisamos apenas criar um objeto Config e então invocar o método Security.fromConfig():

```
Config config = Config.create();
Security security = Security.fromConfig(config);
```

Assim que tivermos a instância Security, primeiro precisamos registrá-la no WebServer usando o método WebSecurity.from():

```
Routing routing = Routing.builder()
  .register(WebSecurity.from(security).securityDefaults(WebSecurity.authenticate()))
  .build();
```

Também podemos criar uma instância WebSecurity diretamente usando a abordagem de configuração pela qual carregamos a segurança e a configuração do servidor web:

```
Routing routing = Routing.builder()        
  .register(WebSecurity.from(config))
  .build();
```

Agora podemos adicionar alguns manipuladores para os caminhos /user e /admin, iniciar o servidor e tentar acessá-los:

```
Routing routing = Routing.builder()
  .register(WebSecurity.from(config))
  .get("/user", (request, response) -> response.send("Hello, I'm Helidon SE"))
  .get("/admin", (request, response) -> response.send("Hello, I'm Helidon SE"))
  .build();
```

# 4. Helidon MP
Helidon MP é uma implementação do Eclipse MicroProfile e também fornece um tempo de execução para a execução de microsserviços baseados em MicroProfile.

Como já temos um artigo sobre Eclipse MicroProfile, vamos verificar esse código-fonte e modificá-lo para rodar no Helidon MP.

Depois de verificar o código, removeremos todas as dependências e plug-ins e adicionaremos as dependências do Helidon MP ao arquivo POM:

```
<dependency>
    <groupId>io.helidon.microprofile.bundles</groupId>
    <artifactId>helidon-microprofile-1.2</artifactId>
    <version>0.10.4</version>
</dependency>
<dependency>
    <groupId>org.glassfish.jersey.media</groupId>
    <artifactId>jersey-media-json-binding</artifactId>
    <version>2.26</version>
</dependency>
```

As dependências helidon-microprofile-1.2 e jersey-media-json-binding estão disponíveis na Maven Central.

A seguir, adicionaremos o arquivo beans.xml no diretório src/main/resource/META-INF com este conteúdo:

```
<beans xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xmlns="http://xmlns.jcp.org/xml/ns/javaee"
  xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
  http://xmlns.jcp.org/xml/ns/javaee/beans_2_0.xsd"
  version="2.0" bean-discovery-mode="annotated">
</beans>
```

Na classe LibraryApplication, substitua o método getClasses() para que o servidor não verifique os recursos:

```
@Override
public Set<Class<?>> getClasses() {
    return CollectionsHelper.setOf(BookEndpoint.class);
}
```

Por fim, crie um método principal e adicione este snippet de código:

```
public static void main(String... args) {
    Server server = Server.builder()
      .addApplication(LibraryApplication.class)
      .port(9080)
      .build();
    server.start();
}
```

E é isso. Agora poderemos invocar todos os recursos do livro.

# 5. Conclusão
Neste artigo, exploramos os principais componentes do Helidon, também mostrando como configurar o Helidon SE e o MP. Como Helidon MP é apenas um tempo de execução Eclipse MicroProfile, podemos executar qualquer microsserviço baseado em MicroProfile existente usando-o.