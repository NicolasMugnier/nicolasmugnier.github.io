# Running PHP on AWS Lambda with Bref and Clean Architecture

## Introduction

AWS Lambda supports custom runtimes, which means you can run virtually any language — including PHP. In this article, we walk through a POC that deploys a PHP 8.1 application to AWS Lambda using **Bref**, a layer-based PHP runtime for Lambda. The application follows **Clean Architecture** principles and manages a simple `Book` resource across three Lambda functions, all wired together with **Symfony Dependency Injection** and deployed via the **Serverless Framework**.

---

## Why PHP on Lambda?

PHP is not natively supported by AWS Lambda. However, Lambda allows you to bring your own runtime via the `provided.al2` execution environment. **Bref** packages a PHP binary as a Lambda Layer, so your function code runs as if PHP were a first-class citizen — no Docker containers, no EC2 instances.

---

## Architecture Overview

```
  Caller (CLI / Event)
         │
         ▼
  ┌─────────────────────────────────────────────┐
  │              AWS Lambda                      │
  │                                             │
  │  ┌─────────────┐  ┌────────────┐  ┌──────────────┐
  │  │  addBook    │  │  getBook   │  │  deleteBook  │
  │  │  handler    │  │  handler   │  │  handler     │
  │  └──────┬──────┘  └─────┬──────┘  └──────┬───────┘
  │         │               │                 │
  │         ▼               ▼                 ▼
  │  ┌─────────────────────────────────────────────┐
  │  │         Symfony DI Container                │
  │  │   AddBook / GetBook / RemoveBook use cases  │
  │  └─────────────────┬───────────────────────────┘
  │                    │
  │                    ▼
  │           ┌─────────────────┐
  │           │  BookRepository  │  (in-memory / pluggable)
  │           └─────────────────┘
  │
  │  Runtime: PHP 8.1 via Bref Layer (provided.al2)
  └─────────────────────────────────────────────┘
```

---

## Project Structure

```
demo/
├── serverless.yml              (infrastructure as code)
├── composer.json
└── src/
    ├── BusinessRules/          (domain — no framework dependencies)
    │   ├── Entities/
    │   │   └── Book.php
    │   ├── Gateways/
    │   │   └── BookGateway.php (interface)
    │   └── UseCases/
    │       ├── AddBook/
    │       │   ├── AddBook.php
    │       │   ├── Request/AddBookRequest.php
    │       │   └── Response/AddBookResponse.php
    │       ├── GetBook/
    │       │   ├── GetBook.php
    │       │   ├── Request/GetBookRequest.php
    │       │   └── Response/GetBookResponse.php
    │       └── RemoveBook/
    │           ├── RemoveBook.php
    │           ├── Request/RemoveBookRequest.php
    │           └── Response/RemoveBookResponse.php
    ├── Repositories/
    │   └── BookRepository.php  (implements BookGateway)
    ├── Events/                 (Lambda entry points)
    │   ├── create/index.php
    │   ├── read/index.php
    │   └── delete/index.php
    └── config/
        └── services.yml        (Symfony DI configuration)
```

The structure is deliberately divided into two worlds: **domain** (`BusinessRules/`) that knows nothing about Lambda, and **infrastructure** (`Events/`, `Repositories/`) that adapts the domain to the outside world.

---

## Infrastructure as Code: `serverless.yml`

All three Lambda functions are declared in a single file:

```yaml
service: app

provider:
  name: aws
  region: eu-west-1
  runtime: provided.al2

plugins:
  - ./vendor/bref/bref

custom:
  bref:
    separateVendor: true

functions:
  addBook:
    handler: src/Events/create/index.php
    layers:
      - ${bref:layer.php-81}

  getBook:
    handler: src/Events/read/index.php
    layers:
      - ${bref:layer.php-81}

  deleteBook:
    handler: src/Events/delete/index.php
    layers:
      - ${bref:layer.php-81}
```

`separateVendor: true` tells Bref to upload the `vendor/` directory as a separate Lambda Layer, keeping each function's deployment package small and fast to update.

---

## Clean Architecture in a Lambda Context

The project applies **Clean Architecture** (also known as Hexagonal Architecture): business rules are isolated at the center, and everything else adapts to them.

### The Entity

```php
// src/BusinessRules/Entities/Book.php
class Book
{
    public function __construct(
        public readonly string $id,
        public readonly string $title,
        public readonly string $author,
    ) {}
}
```

### The Gateway Interface

```php
// src/BusinessRules/Gateways/BookGateway.php
interface BookGateway
{
    public function add(Book $book): Book;
    public function findById(string $id): ?Book;
    public function remove(string $id): void;
}
```

`BookGateway` is a pure PHP interface. The business rules layer never depends on DynamoDB, MySQL, or any other storage detail.

### A Use Case

```php
// src/BusinessRules/UseCases/AddBook/AddBook.php
class AddBook implements UseCase
{
    public function __construct(private BookGateway $bookGateway) {}

    public function execute(UseCaseRequest $request): UseCaseResponse
    {
        $book = new Book(uniqid(), $request->title, $request->author);
        $this->bookGateway->add($book);
        return AddBookResponse::create($book);
    }
}
```

Use cases are plain PHP classes. They receive a typed Request DTO, perform domain logic, and return a typed Response DTO — no HTTP, no Lambda, no framework.

### The Repository (Adapter)

```php
// src/Repositories/BookRepository.php
class BookRepository implements BookGateway
{
    private array $books = [];

    public function add(Book $book): Book
    {
        $this->books[$book->id] = $book;
        return $book;
    }

    public function findById(string $id): ?Book
    {
        return $this->books[$id] ?? null;
    }

    public function remove(string $id): void
    {
        unset($this->books[$id]);
    }
}
```

Today this is in-memory storage — enough for a demo. Swapping it for a DynamoDB repository only requires implementing `BookGateway` with a different class and updating the DI binding in `services.yml`. The use cases don't change.

---

## Lambda Handlers

Each handler is the Lambda entry point. Its only jobs are: bootstrap the DI container, parse the event, run the use case, and return a response.

```php
// src/Events/create/index.php
require '/tmp/vendor/autoload.php';

$container = new ContainerBuilder();
(new YamlFileLoader($container, new FileLocator(__DIR__ . '/../../config/')))->load('services.yml');
$container->compile();

return function (array $event) use ($container): array {
    $request = AddBookRequest::create($event['title'], $event['author']);
    /** @var AddBook $useCase */
    $useCase = $container->get(AddBook::class);
    $response = $useCase->execute($request);

    return ['id' => $response->book->id];
};
```

Note the `/tmp/vendor/autoload.php` path: Lambda's ephemeral filesystem mounts the separate vendor layer under `/tmp`, which is why Bref's `separateVendor` option writes the autoloader there.

---

## Dependency Injection with Symfony

All services are declared in `config/services.yml`:

```yaml
services:
  _defaults:
    autowire: true
    autoconfigure: true

  App\:
    resource: '../../src/*'
    exclude: '../../src/Events'

  App\BusinessRules\Gateways\BookGateway:
    class: App\Repositories\BookRepository

  App\BusinessRules\UseCases\AddBook\AddBook:
    public: true

  App\BusinessRules\UseCases\GetBook\GetBook:
    public: true

  App\BusinessRules\UseCases\RemoveBook\RemoveBook:
    public: true
```

`autowire: true` means Symfony resolves constructor dependencies automatically. The only explicit binding is `BookGateway → BookRepository`, which is exactly the Dependency Inversion Principle in action.

---

## Invoking the Functions

```bash
# Deploy
composer install
serverless deploy

# Add a book
serverless invoke --function addBook -d '{"title": "Clean Code", "author": "Robert C. Martin"}'
# → {"id": "64a1bc..."}

# Get a book
serverless invoke --function getBook -d '{"id": "64a1bc..."}'
# → {"id": "64a1bc...", "title": "Clean Code", "author": "Robert C. Martin"}

# Delete a book
serverless invoke --function deleteBook -d '{"id": "64a1bc..."}'
# → null
```

---

## Key Patterns Illustrated

| Pattern | Implementation |
|---------|---------------|
| **Custom Lambda runtime** | Bref `php-81` layer on `provided.al2` |
| **Clean Architecture** | Domain isolated from Lambda, Symfony, and storage |
| **Dependency Inversion** | `BookGateway` interface decouples use cases from repository |
| **Request/Response DTOs** | Typed input/output for every use case |
| **Dependency Injection** | Symfony DI container bootstrapped per invocation |
| **Separate vendor layer** | Faster deploys with `separateVendor: true` |
| **Strict types** | `declare(strict_types=1)` enforced across all PHP files |

---

## Going Further

This POC is intentionally minimal. To move toward production, you would:

1. **Add a real database** — Implement a `DynamoDbBookRepository` that implements `BookGateway` and interacts with DynamoDB via `async-aws/dynamodb`. No use case code changes required.
2. **Add an HTTP trigger** — Attach an API Gateway event to each function and map HTTP methods to the right handlers.
3. **Add IAM permissions** — Use `serverless-iam-roles-per-function` to grant each function only the DynamoDB actions it needs.
4. **Add tests** — Use cases are plain PHP classes, easy to unit test without mocking Lambda or AWS.
5. **Set up CI/CD** — A GitHub Actions workflow with OIDC federation can run `serverless deploy` on every push to `main`.

---

## Conclusion

This demo challenges the assumption that PHP and serverless don't mix. With Bref, running PHP 8.1 on Lambda is straightforward. The key takeaways:

- **Bref** makes PHP a first-class Lambda runtime with zero Docker overhead
- **Clean Architecture** pays off in a Lambda context: business logic is testable, portable, and completely decoupled from the event source
- **Symfony DI** works perfectly in Lambda — lightweight enough to bootstrap per invocation without performance issues
- **Serverless Framework** turns multi-function infrastructure into a few lines of YAML
- **Swapping the repository** (in-memory → DynamoDB → any database) requires no changes to domain code — only a new implementation of `BookGateway`
