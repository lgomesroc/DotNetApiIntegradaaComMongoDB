# Criar o projeto da API Web do ASP.NET Core

- Definir uma coleção e um esquema do MongoDB
- Executar operações CRUD do MongoDB a partir de uma API Web
- Personalizar a serialização JSON

Execute os seguintes comandos em um shell de comando:

*CLI do .NET*

**dotnet new webapi -o BooksApi**
**code BooksApi**

Um novo projeto de API Web do ASP.NET Core direcionado ao .NET Core é gerado e aberto no Visual Studio Code.

Depois que o ícone de chama de OmniSharp da barra de status fica verde, uma caixa de diálogo solicita que os ativos necessários compilem e depurem estão faltando em ' BooksApi '. Adicioná-los?. Selecione Sim na barra superior.

Visite a Galeria do NuGet: MongoDB. Driver para determinar a versão estável mais recente do driver .net para MongoDB. Abra Terminal Integrado e navegue até a raiz do projeto. Execute o seguinte comando para instalar o driver .NET para MongoDB:

*CLI do .NET*

**dotnet add BooksApi.csproj package MongoDB.Driver -v {VERSION}**

## Adicionar um modelo de entidade

Adicione um diretório Modelos à raiz do projeto.

Adicione uma classe Book ao diretório Modelos com o seguinte código:

### C#

using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;

namespace BooksApi.Models{
    public class Book    {
        [BsonId]
        [BsonRepresentation(BsonType.ObjectId)]
        public string Id { get; set; }

        [BsonElement("Name")]
        public string BookName { get; set; }

        public decimal Price { get; set; }

        public string Category { get; set; }

        public string Author { get; set; }
    }
}

Na classe anterior, a propriedade Id:

É necessária para mapear o objeto CLR (Common Language Runtime) para a coleção do MongoDB.

É anotada com [BsonId] para designar essa propriedade como a chave primária do documento.

É anotado com [BsonRepresentation(BsonType.ObjectId)] para permitir a passagem do parâmetro como tipo string em vez de uma estrutura ObjectID . O Mongo processa a conversão de string para ObjectId.

A BookName propriedade é anotada com o [BsonElement] atributo. O valor do atributo de Name representa o nome da propriedade da coleção do MongoDB.

## Adicionar um modelo de configuração

Adicione os seguintes valores de configuração de banco de dados a appsettings.json :

*JSON*

{
  "BookstoreDatabaseSettings": {
    "BooksCollectionName": "Books",
    "ConnectionString": "mongodb://localhost:27017",
    "DatabaseName": "BookstoreDb"
  },
  "Logging": {
    "IncludeScopes": false,
    "Debug": {
      "LogLevel": {
        "Default": "Warning"
      }
    },
    "Console": {
      "LogLevel": {
        "Default": "Warning"
      }
    }
  }
}

Adicione um arquivo BookstoreDatabaseSettings.cs no diretório Modelos com o código a seguir:

*C#*

namespace BooksApi.Models{
    public class BookstoreDatabaseSettings : IBookstoreDatabaseSettings    {
        public string BooksCollectionName { get; set; }
        public string ConnectionString { get; set; }
        public string DatabaseName { get; set; }
    }

    public interface IBookstoreDatabaseSettings    {
        string BooksCollectionName { get; set; }
        string ConnectionString { get; set; }
        string DatabaseName { get; set; }
    }
}

A BookstoreDatabaseSettings classe anterior é usada para armazenar os appsettings.json valores de BookstoreDatabaseSettings Propriedade do arquivo. Os nomes de propriedade JSON e C# são nomeados de forma idêntica para facilitar o processo de mapeamento.

Adicione o código realçado a seguir a Startup.ConfigureServices:

*C#*

public void ConfigureServices(IServiceCollection services){
    // requires using Microsoft.Extensions.Options
    services.Configure<BookstoreDatabaseSettings>(
        Configuration.GetSection(nameof(BookstoreDatabaseSettings)));

    services.AddSingleton<IBookstoreDatabaseSettings>(sp =>
        sp.GetRequiredService<IOptions<BookstoreDatabaseSettings>>().Value);

    services.AddControllers();
}

No código anterior:

A instância de configuração à qual a appsettings.json seção do arquivo BookstoreDatabaseSettings é associada é registrada no contêiner injeção de dependência (di). Por exemplo, a BookstoreDatabaseSettings propriedade de um objeto ConnectionString é populada com a BookstoreDatabaseSettings:ConnectionString propriedade em appsettings.json .

A interface IBookstoreDatabaseSettings é registrada na DI com um tempo de vida do serviço singleton. Quando inserida, a instância da interface é resolvida para um objeto BookstoreDatabaseSettings.

Adicione o seguinte código na parte superior do Startup.cs para resolver as referências BookstoreDatabaseSettings e IBookstoreDatabaseSettings:

*C#*

**using BooksApi.Models;**

Adicionar um serviço de operações CRUD

Adicione um diretório Serviços à raiz do projeto.

Adicione uma classe BookService ao diretório Serviços com o seguinte código:

*C#*

using BooksApi.Models;
using MongoDB.Driver;
using System.Collections.Generic;
using System.Linq;

namespace BooksApi.Services{
    public class BookService    {
        private readonly IMongoCollection<Book> _books;

        public BookService(IBookstoreDatabaseSettings settings)        {
            var client = new MongoClient(settings.ConnectionString);
            var database = client.GetDatabase(settings.DatabaseName);

            _books = database.GetCollection<Book>(settings.BooksCollectionName);
        }

        public List<Book> Get() =>
            _books.Find(book => true).ToList();

        public Book Get(string id) =>
            _books.Find<Book>(book => book.Id == id).FirstOrDefault();

        public Book Create(Book book)        {
            _books.InsertOne(book);
            return book;
        }

        public void Update(string id, Book bookIn) =>
            _books.ReplaceOne(book => book.Id == id, bookIn);

        public void Remove(Book bookIn) =>
            _books.DeleteOne(book => book.Id == bookIn.Id);

        public void Remove(string id) => 
            _books.DeleteOne(book => book.Id == id);
    }
}

No código anterior, uma instância IBookstoreDatabaseSettings é recuperada da DI por meio da injeção de construtor. Essa técnica fornece acesso aos appsettings.json valores de configuração que foram adicionados na seção Adicionar um modelo de configuração .

Adicione o código realçado a seguir a Startup.ConfigureServices:

*C#*

public void ConfigureServices(IServiceCollection services){
    services.Configure<BookstoreDatabaseSettings>(
        Configuration.GetSection(nameof(BookstoreDatabaseSettings)));

    services.AddSingleton<IBookstoreDatabaseSettings>(sp =>
        sp.GetRequiredService<IOptions<BookstoreDatabaseSettings>>().Value);

    services.AddSingleton<BookService>();

    services.AddControllers();
}

No código anterior, a classe BookService é registrada com a DI para dar suporte à injeção de construtor nas classes consumidoras. O tempo de vida do serviço singleton é mais apropriado porque BookService usa uma dependência direta de MongoClient. De acordo com as diretrizes oficiais de reutilizaçãodo Cliente Mongo, deve ser registrado na MongoClient DI com um tempo de vida de serviço singleton.

Adicione o seguinte código na parte superior do Startup.cs para resolver a referências BookService:

*C#*

**using BooksApi.Services;**

A classe BookService usa os seguintes membros MongoDB.Driver para executar operações CRUD em relação ao banco de dados:

MongoClient:lê a instância do servidor para executar operações de banco de dados. O construtor dessa classe é fornecido na cadeia de conexão do MongoDB:

*C#*

public BookService(IBookstoreDatabaseSettings settings){
    var client = new MongoClient(settings.ConnectionString);
    var database = client.GetDatabase(settings.DatabaseName);

    _books = database.GetCollection<Book>(settings.BooksCollectionName);
}

- IMongoDatabase:representa o banco de dados Mongo para executar operações. Este tutorial usa o método GetCollection<TDocument>(collection) genérico na interface para obter acesso aos dados em uma coleção específica. Execute operações CRUD em relação à coleção depois que esse método for chamado. Na chamada de método GetCollection<TDocument>(collection):

- collection representa o nome da coleção.

- TDocument representa o tipo de objeto CLR armazenado na coleção.

- GetCollection<TDocument>(collection) retorna um objeto MongoCollection que representa a coleção. Neste tutorial, os seguintes métodos são invocados na coleção:

- DeleteOne:exclui um único documento que corresponde aos critérios de pesquisa fornecidos.

- Find <TDocument> : retorna todos os documentos na coleção que corresponde aos critérios de pesquisa fornecidos.

- InsertOne:insere o objeto fornecido como um novo documento na coleção.

- ReplaceOne:substitui o único documento que corresponde aos critérios de pesquisa fornecidos pelo objeto fornecido.

### Adicionar um controlador

Adicione uma classe BooksController ao diretório Controladores com o seguinte código:

*C#*

using BooksApi.Models;
using BooksApi.Services;
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;

namespace BooksApi.Controllers {
    [Route("api/[controller]")]
    [ApiController]
    public class BooksController : ControllerBase    {
        private readonly BookService _bookService;

        public BooksController(BookService bookService)        {
            _bookService = bookService;
        }

        [HttpGet]
        public ActionResult<List<Book>> Get() =>
            _bookService.Get();

        [HttpGet("{id:length(24)}", Name = "GetBook")]
        public ActionResult<Book> Get(string id) {
            var book = _bookService.Get(id);

            if (book == null)        {
                return NotFound();
            }

            return book;
        }

        [HttpPost]
        public ActionResult<Book> Create(Book book)                {
            _bookService.Create(book);

            return CreatedAtRoute("GetBook", new { id = book.Id.ToString() }, book);
        }

        [HttpPut("{id:length(24)}")]
        public IActionResult Update(string id, Book bookIn)        {
            var book = _bookService.Get(id);

            if (book == null)            {
                return NotFound();
            }

            _bookService.Update(id, bookIn);

            return NoContent();
        }

        [HttpDelete("{id:length(24)}")]
        public IActionResult Delete(string id)        {
            var book = _bookService.Get(id);

            if (book == null)            {
                return NotFound();
            }

            _bookService.Remove(book.Id);

            return NoContent();
        }
    }
}

### O controlador da API Web anterior:

Usa a classe BookService para executar operações CRUD.

Contém métodos de ação para dar suporte a solicitações GET, POST, PUT e DELETE HTTP.

Chama o CreatedAtRoute no método de ação Create para retornar uma resposta HTTP 201. O código de status 201 é a resposta padrão para um método HTTP POST que cria um recurso no servidor. CreatedAtRoute também adiciona um cabeçalho Location à resposta. O cabeçalho Location especifica o URI do livro recém-criado.

### Testar a API Web

Compile e execute o aplicativo.

Navegue até http://localhost:<port>/api/books para testar o método de ação Get sem parâmetros do controlador. A seguinte resposta JSON é exibida:

*JSON*

[
  {
    "id":"5bfd996f7b8e48dc15ff215d",
    "bookName":"Design Patterns",
    "price":54.93,
    "category":"Computers",
    "author":"Ralph Johnson"
  },
  {
    "id":"5bfd996f7b8e48dc15ff215e",
    "bookName":"Clean Code",
    "price":43.15,
    "category":"Computers",
    "author":"Robert C. Martin"
  }
]

Navegue até http://localhost:<port>/api/books/{id here} para testar o método de ação Get sobrecarregado do controlador. A seguinte resposta JSON é exibida:

*JSON*

{
  "id":"{ID}",
  "bookName":"Clean Code",
  "price":43.15,
  "category":"Computers",
  "author":"Robert C. Martin"
}

### Configurar opções de serialização JSON

Há dois detalhes alterar sobre as respostas JSON retornadas na seção Testar a API Web:

O uso de maiúsculas e minúsculas padrão dos nomes da propriedade deve ser alterado para corresponder ao uso de maiúsculas e minúsculas Pascal dos nomes de propriedade do objeto CLR.

A propriedade bookName deve ser retornada como Name.

Para cumprir os requisitos anteriores, faça as seguintes alterações:

O JSON.NET foi removido da estrutura compartilhada do ASP.NET. Adicione uma referência de pacote a Microsoft.AspNetCore.Mvc.NewtonsoftJson .

Em Startup.ConfigureServices, encadeie o seguinte código realçado para a chamada de método AddControllers:

*C#*

public void ConfigureServices(IServiceCollection services){
    services.Configure<BookstoreDatabaseSettings>(
        Configuration.GetSection(nameof(BookstoreDatabaseSettings)));

    services.AddSingleton<IBookstoreDatabaseSettings>(sp =>
        sp.GetRequiredService<IOptions<BookstoreDatabaseSettings>>().Value);

    services.AddSingleton<BookService>();

    services.AddControllers()
        .AddNewtonsoftJson(options => options.UseMemberCasing());
}

Com a alteração anterior, os nomes de propriedade na resposta JSON serializada da API Web correspondem aos respectivos nomes de propriedade no tipo de objeto CLR. Por exemplo, a propriedade Author da classe Book é serializada como Author.

Em Models/Book.cs, anote a BookName propriedade com o seguinte [JsonProperty] atributo:

*C#*

Copiar
[BsonElement("Name")]
[JsonProperty("Name")]
public string BookName { get; set; }
O valor do atributo [JsonProperty] de Name representa o nome da propriedade da resposta JSON serializada da API Web.

Adicione o seguinte código à parte superior de Models/Book.cs para resolver a referência de atributo [JsonProperty]:

C#

Copiar
using Newtonsoft.Json;
Repita as etapas definidas na seção Testar a API Web. Observe a diferença em nomes de propriedade JSON.

Adicionar suporte de autenticação a uma API da Web
ASP.NET Core Identity adiciona a funcionalidade de logon da interface do usuário aos ASP.NET Web Core. Para proteger APIs Web e SPAs, use um dos seguintes:

Azure Active Directory
Azure Active Directory B2C (Azure AD B2C)
IdentityServer4
IdentityServer4 é uma estrutura OpenID Connect e OAuth 2.0 para ASP.NET Core. IdentityO Server4 habilita os seguintes recursos de segurança:

Autenticação como serviço (AaaS)
SSO (single sign-on/off) em vários tipos de aplicativos
Controle de acesso para APIs
Gateway de Federação
Para obter mais informações, consulte Bem-vindo ao Identity Server4.


## erro 

* Instalar*
**dotnet add package Microsoft.AspNetCore.Mvc.NewtonsoftJson --version 3.1.0**
