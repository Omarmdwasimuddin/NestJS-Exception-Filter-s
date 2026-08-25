## Exception Filters

### Create filter
```bash
nest g filter filters/http-exception
```
---

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
