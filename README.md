## Exception Filters

### Create filter
```bash
nest g filter filters/http-exception
```

<img width="318" height="67" alt="image" src="https://github.com/user-attachments/assets/5c31cf7e-fd26-4724-858a-96927bf6f0de" />

### `http-exception.filter.ts`
```bash
import { ArgumentsHost, Catch, ExceptionFilter, HttpException } from '@nestjs/common';
import { Response, Request } from 'express';


@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();

    response.status(status).json({
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      message: exception.message
    });
  }
}
```
---



### Create controller
```bash
nest g controller exception
```
<img width="302" height="71" alt="image" src="https://github.com/user-attachments/assets/2b476713-f797-4bc7-a2b8-9b093a40219e" />

### `exception.controller.ts`
```bash
import { Controller, Get, Param, ParseIntPipe, UseFilters } from '@nestjs/common';
import { HttpExceptionFilter } from 'src/filters/http-exception/http-exception.filter';

@Controller('exception')
@UseFilters(HttpExceptionFilter)
export class ExceptionController {
    @Get('hello/:id')
    getHello(@Param('id', ParseIntPipe) id: number) {
        return {message: `Your ID is ${id}`}
    }
}
```
---


> ## Note: id must be number dite hobe
>
> <img width="400" height="186" alt="image" src="https://github.com/user-attachments/assets/7ef3c8c4-05b3-47c5-9380-8e5527b7bfb6" />
>
> Note: string dile id show hobe na
>
> <img width="458" height="206" alt="image" src="https://github.com/user-attachments/assets/16e1df4b-fb2d-4155-97eb-e68afb1286af" />
