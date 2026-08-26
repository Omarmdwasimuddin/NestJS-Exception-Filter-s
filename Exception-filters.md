# NestJS Exception Filters — সহজ বাংলা ব্যাখ্যা

## Exception Filter কী?

Nest-এর একটা built-in "exceptions layer" আছে, যেটা পুরো application-এ যত unhandled exception হয় সব handle করে। মানে তোমার code-এ যদি কোনো exception throw হয় আর সেটাকে ধরার (catch) মতো কিছু না থাকে, তাহলে এই layer সেটা ধরে ফেলে এবং automatic ভাবে একটা user-friendly response পাঠিয়ে দেয়।

By default এই কাজটা করে একটা **built-in global exception filter**, যেটা `HttpException` (এবং তার subclass) type-এর exception handle করে। যদি কোনো exception `HttpException`-এর না হয় (অচেনা exception), তাহলে default এই response আসে:

```json
{
  "statusCode": 500,
  "message": "Internal server error"
}
```

> **Hint:** Global exception filter আংশিকভাবে `http-errors` library-ও support করে — মানে কোনো exception-এ যদি `statusCode` আর `message` property থাকে, সেগুলো সঠিকভাবে response-এ বসে যাবে (অচেনা exception-এর জন্য default `InternalServerErrorException` না দিয়ে)।

---

## Standard Exception Throw করা

Nest-এর `@nestjs/common` package থেকে একটা built-in `HttpException` class পাওয়া যায়। সাধারণ HTTP REST/GraphQL API-তে error হলে standard HTTP response object পাঠানোই best practice।

```ts
@Get()
async findAll() {
  throw new HttpException('Forbidden', HttpStatus.FORBIDDEN);
}
```

এই route call করলে response হবে:

```json
{
  "statusCode": 403,
  "message": "Forbidden"
}
```

### HttpException constructor-এর argument

`HttpException`-এর constructor-এ ৩টা argument থাকে:

1. **response** — JSON response body ঠিক করে। এটা string বা object দুটোই হতে পারে।
   - string দিলে শুধু `message` অংশটা override হয়
   - object দিলে পুরো response body-ই override হয়ে যায়
2. **status** — HTTP status code (best practice হলো `HttpStatus` enum ব্যবহার করা)
3. **options** *(optional)* — এখানে `cause` দেওয়া যায়, যেটা response-এ যায় না কিন্তু logging-এর জন্য কাজে লাগে (আসল error কোথা থেকে এলো সেটা বোঝার জন্য)

পুরো response body override করার উদাহরণ:

```ts
@Get()
async findAll() {
  try {
    await this.service.findAll()
  } catch (error) {
    throw new HttpException({
      status: HttpStatus.FORBIDDEN,
      error: 'This is a custom message',
    }, HttpStatus.FORBIDDEN, {
      cause: error
    });
  }
}
```

Response হবে:

```json
{
  "status": 403,
  "error": "This is a custom message"
}
```

---

## Exception Logging

By default, `HttpException` (আর তার subclass) built-in filter automatically console-এ log করে **না** — কারণ এগুলোকে normal application flow-এর অংশ ধরা হয়। একই আচরণ `WsException` আর `RpcException`-এর ক্ষেত্রেও হয়।

এই সবগুলো exception `IntrinsicException` নামের base class থেকে inherit করে (from `@nestjs/common`), যেটা দিয়ে normal flow-এর exception আর আসল bug-এর exception আলাদা করা হয়।

যদি এগুলো log করতে চাও, নিজের custom exception filter বানাতে হবে (নিচে দেখানো হয়েছে)।

---

## Custom Exception বানানো

বেশিরভাগ ক্ষেত্রে built-in HTTP exception-ই যথেষ্ট। কিন্তু নিজের custom exception দরকার হলে, `HttpException`-কে extend করে নিজের exception hierarchy বানানো ভালো practice — এতে Nest সেটাকে চিনবে এবং error response automatic handle করবে।

```ts
export class ForbiddenException extends HttpException {
  constructor() {
    super('Forbidden', HttpStatus.FORBIDDEN);
  }
}
```

যেহেতু এটা `HttpException`-কে extend করেছে, এটা built-in exception handler-এর সাথে নিজে থেকেই কাজ করবে:

```ts
@Get()
async findAll() {
  throw new ForbiddenException();
}
```

---

## Built-in HTTP Exception-এর তালিকা

`@nestjs/common` থেকে অনেকগুলো standard exception পাওয়া যায় (সবগুলোই `HttpException`-এর child):

`BadRequestException`, `UnauthorizedException`, `NotFoundException`, `ForbiddenException`, `NotAcceptableException`, `RequestTimeoutException`, `ConflictException`, `GoneException`, `HttpVersionNotSupportedException`, `PayloadTooLargeException`, `UnsupportedMediaTypeException`, `UnprocessableEntityException`, `InternalServerErrorException`, `NotImplementedException`, `ImATeapotException`, `MethodNotAllowedException`, `BadGatewayException`, `ServiceUnavailableException`, `GatewayTimeoutException`, `PreconditionFailedException`

এগুলোতেও `options` parameter দিয়ে `cause` আর `description` দেওয়া যায়:

```ts
throw new BadRequestException('Something bad happened', {
  cause: new Error(),
  description: 'Some error description',
});
```

Response:

```json
{
  "message": "Something bad happened",
  "error": "Some error description",
  "statusCode": 400
}
```

---

## নিজের Exception Filter বানানো

Built-in filter অনেক কিছু automatic handle করে দেয়, কিন্তু নিজের control চাইলে (যেমন — logging যোগ করা, dynamic factor অনুযায়ী আলাদা JSON schema দেওয়া) নিজের **exception filter** বানানো যায়।

`HttpException`-এর instance catch করে custom response logic লেখার উদাহরণ — এখানে underlying `Request`/`Response` object access করতে হবে:

```ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response
      .status(status)
      .json({
        statusCode: status,
        timestamp: new Date().toISOString(),
        path: request.url,
      });
  }
}
```

> **Hint:** সব exception filter-কেই generic `ExceptionFilter<T>` interface implement করতে হয় — মানে `catch(exception: T, host: ArgumentsHost)` method থাকতে হবে। `T` হলো exception-এর type।

> **Warning (Fastify ব্যবহারকারীদের জন্য):** `@nestjs/platform-fastify` ব্যবহার করলে `response.json()`-এর বদলে `response.send()` ব্যবহার করতে হবে, এবং fastify থেকে সঠিক type import করতে হবে।

`@Catch(HttpException)` decorator দিয়ে বলে দেওয়া হয় এই filter শুধু `HttpException` type-এর exception-ই খুঁজবে। `@Catch()`-এ একাধিক exception type comma দিয়ে দেওয়া যায়, তাহলে একটা filter একাধিক exception type handle করতে পারবে।

### ArgumentsHost কী?

`catch()` method-এর দুইটা parameter:
- **exception** — যে exception object process হচ্ছে
- **host** — একটা `ArgumentsHost` object, যেটা দিয়ে original request handler-এ পাঠানো `Request`/`Response` object পাওয়া যায়

`ArgumentsHost` সব context-এই কাজ করে (HTTP, Microservices, WebSockets) — এই abstraction-এর কারণেই একটাই generic exception filter সব ধরনের context-এ ব্যবহার করা যায়।

---

## Filter Bind করা (কোথায় apply হবে)

### Method-scoped

```ts
@Post()
@UseFilters(new HttpExceptionFilter())
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

Instance না দিয়ে class-ও দেওয়া যায় (তখন Nest নিজে instantiate করবে, DI-ও কাজ করবে):

```ts
@Post()
@UseFilters(HttpExceptionFilter)
async create(@Body() createCatDto: CreateCatDto) {
  throw new ForbiddenException();
}
```

> **Hint:** যেখানে সম্ভব instance-এর বদলে class দিয়ে filter apply করাই ভালো — এতে memory কম লাগে, কারণ Nest পুরো module জুড়ে একই class-এর instance reuse করতে পারে।

### Controller-scoped

```ts
@Controller()
@UseFilters(new HttpExceptionFilter())
export class CatsController {}
```

এতে `CatsController`-এর ভিতরের সবগুলো route handler-এ filter apply হয়ে যায়।

### Global-scoped

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.useGlobalFilters(new HttpExceptionFilter());
  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

> **Warning:** `useGlobalFilters()` gateway বা hybrid application-এর জন্য কাজ করে না।

**সমস্যা:** এভাবে (`useGlobalFilters()` দিয়ে, module-এর বাইরে থেকে) global filter বানালে সেটাতে DI দিয়ে dependency inject করা যায় না, কারণ এটা কোনো module-এর context-এর বাইরে থেকে register হচ্ছে।

**সমাধান:** `APP_FILTER` token ব্যবহার করে module-এর ভিতর থেকেই global filter register করা:

```ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';

@Module({
  providers: [
    {
      provide: APP_FILTER,
      useClass: HttpExceptionFilter,
    },
  ],
})
export class AppModule {}
```

> **Hint:** এভাবে register করলে filter-টা global-ই থাকে, তা যেই module-এই লেখা হোক না কেন। যে module-এ filter define করা আছে, সেখানেই এটা লেখা ভালো। `useClass` ছাড়াও provider register করার আরও উপায় আছে।

এই পদ্ধতিতে চাইলে যতগুলো ইচ্ছা filter `providers` array-এ যোগ করা যায়।

---

## সবকিছু Catch করা (Catch Everything)

সব ধরনের unhandled exception ধরতে চাইলে (type নির্বিশেষে), `@Catch()`-এর parameter list খালি রাখতে হবে — শুধু `@Catch()`।

Platform-agnostic (Express/Fastify দুটোতেই কাজ করবে) উদাহরণ — এখানে সরাসরি `Request`/`Response` object না ব্যবহার করে `HttpAdapter` ব্যবহার করা হয়েছে:

```ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { HttpAdapterHost } from '@nestjs/core';

@Catch()
export class CatchEverythingFilter implements ExceptionFilter {
  constructor(private readonly httpAdapterHost: HttpAdapterHost) {}

  catch(exception: unknown, host: ArgumentsHost): void {
    const { httpAdapter } = this.httpAdapterHost;

    const ctx = host.switchToHttp();

    const httpStatus =
      exception instanceof HttpException
        ? exception.getStatus()
        : HttpStatus.INTERNAL_SERVER_ERROR;

    const responseBody = {
      statusCode: httpStatus,
      timestamp: new Date().toISOString(),
      path: httpAdapter.getRequestUrl(ctx.getRequest()),
    };

    httpAdapter.reply(ctx.getResponse(), responseBody, httpStatus);
  }
}
```

> **Warning:** যদি "catch everything" filter আর কোনো নির্দিষ্ট type-এর filter একসাথে ব্যবহার করো, তাহলে "catch everything" filter-টা আগে declare করতে হবে — যাতে নির্দিষ্ট type-এর filter সঠিকভাবে তার bound type handle করতে পারে।

---

## Inheritance (Base Filter Extend করা)

সবসময় পুরোপুরি custom filter না বানিয়ে, built-in default global exception filter-কে extend করেও কাজ চালানো যায় — শুধু নির্দিষ্ট কিছু জায়গায় behavior override করে।

এর জন্য `BaseExceptionFilter` extend করে ভেতরের `catch()` method call করতে হয়:

```ts
import { Catch, ArgumentsHost } from '@nestjs/common';
import { BaseExceptionFilter } from '@nestjs/core';

@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    super.catch(exception, host);
  }
}
```

> **Warning:** `BaseExceptionFilter` extend করা Method-scoped বা Controller-scoped filter কখনো `new` দিয়ে নিজে instantiate করা উচিত না — Nest framework-কেই automatic instantiate করতে দিতে হবে।

Global filter-ও base filter extend করতে পারে, দুইভাবে:

**পদ্ধতি ১:** custom global filter বানানোর সময় `HttpAdapter` inject করা:

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const { httpAdapter } = app.get(HttpAdapterHost);
  app.useGlobalFilters(new AllExceptionsFilter(httpAdapter));

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

**পদ্ধতি ২:** `APP_FILTER` token ব্যবহার করা (উপরে দেখানো হয়েছে)।

---
| Inheritance | `BaseExceptionFilter` extend করে default behavior-এর উপর build করা যায় |
