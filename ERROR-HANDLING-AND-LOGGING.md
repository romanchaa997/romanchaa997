# Error Handling and Logging Guide

## Executive Summary
Comprehensive error handling and logging strategy ensuring robust error management, detailed diagnostics, and complete audit trails across all services.

## Part 1: Error Handling Principles

### 1.1 Error Hierarchy

```python
class ApplicationError(Exception):
    """Base application error"""
    def __init__(self, code, message, status_code=500, details=None):
        self.code = code
        self.message = message
        self.status_code = status_code
        self.details = details or {}
        super().__init__(message)

class ValidationError(ApplicationError):
    """Input validation errors"""
    def __init__(self, message, field=None, value=None):
        super().__init__(
            code='VALIDATION_ERROR',
            message=message,
            status_code=422,
            details={'field': field, 'value': value}
        )

class BusinessLogicError(ApplicationError):
    """Business logic violations"""
    def __init__(self, message, reason=None):
        super().__init__(
            code='BUSINESS_ERROR',
            message=message,
            status_code=400,
            details={'reason': reason}
        )

class ResourceNotFoundError(ApplicationError):
    """Resource not found errors"""
    def __init__(self, resource_type, resource_id):
        super().__init__(
            code='NOT_FOUND',
            message=f'{resource_type} not found',
            status_code=404,
            details={
                'resource_type': resource_type,
                'resource_id': resource_id
            }
        )

class ExternalServiceError(ApplicationError):
    """External service failures"""
    def __init__(self, service_name, error_message):
        super().__init__(
            code='EXTERNAL_SERVICE_ERROR',
            message=f'{service_name} service error',
            status_code=502,
            details={
                'service': service_name,
                'error': error_message
            }
        )
```

### 1.2 Error Handling Patterns

#### Try-Catch Pattern
```python
try:
    user = get_user(user_id)
    orders = get_user_orders(user.id)
except ResourceNotFoundError as e:
    logger.error(f'User not found: {user_id}')
    return error_response(e.code, e.message, e.status_code)
except ExternalServiceError as e:
    logger.error(f'External service error: {e.details}')
    return error_response(e.code, e.message, 503)
except Exception as e:
    logger.exception(f'Unexpected error: {str(e)}')
    return error_response('INTERNAL_ERROR', 'An unexpected error occurred', 500)
```

#### Context Manager Pattern
```python
from contextlib import contextmanager

@contextmanager
def handle_errors():
    try:
        yield
    except ValidationError as e:
        logger.warning(f'Validation error: {e.message}')
        raise
    except ApplicationError as e:
        logger.error(f'Application error: {e.code}')
        raise
    except Exception as e:
        logger.exception('Unexpected error')
        raise ApplicationError(
            code='INTERNAL_ERROR',
            message='An unexpected error occurred'
        )

# Usage
with handle_errors():
    process_user_order(user_id, order_data)
```

#### Async Error Handling
```python
async def fetch_user_data(user_id):
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get(f'/users/{user_id}')
            response.raise_for_status()
            return response.json()
    except httpx.HTTPError as e:
        raise ExternalServiceError('UserService', str(e))
    except Exception as e:
        raise ApplicationError('FETCH_ERROR', f'Failed to fetch user: {str(e)}')
```

## Part 2: Logging Strategy

### 2.1 Log Levels

| Level | Use Case | Example |
|-------|----------|----------|
| DEBUG | Detailed diagnostic info | Variable values, function entry/exit |
| INFO | General informational messages | User login, order created |
| WARNING | Warning conditions | Deprecation, unusual but handled conditions |
| ERROR | Error events that may allow recovery | Failed database query, missing resource |
| CRITICAL | Critical errors requiring immediate action | Service down, data loss risk |

### 2.2 Structured Logging

#### Python with Python-JSON-Logger

```python
import logging
import json
from pythonjsonlogger import jsonlogger

def setup_logging():
    logger = logging.getLogger()
    logHandler = logging.StreamHandler()
    formatter = jsonlogger.JsonFormatter()
    logHandler.setFormatter(formatter)
    logger.addHandler(logHandler)
    logger.setLevel(logging.INFO)
    return logger

logger = setup_logging()

# Logging with context
logger.info('User created', extra={
    'user_id': 123,
    'email': 'user@example.com',
    'event': 'user.created',
    'timestamp': '2024-01-03T05:15:00Z'
})
```

#### Node.js with Winston

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  defaultMeta: { service: 'api-server' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}

// Usage
logger.info('User created', {
  userId: 123,
  email: 'user@example.com',
  requestId: req.id
});
```

### 2.3 Log Context

Always include:
- Request ID (for tracing)
- User ID (when applicable)
- Timestamp (ISO 8601)
- Service name
- Environment (dev/staging/prod)
- Version

```json
{
  "timestamp": "2024-01-03T05:15:00Z",
  "level": "INFO",
  "service": "api-backend",
  "environment": "production",
  "version": "v1.2.3",
  "request_id": "req-12345abc",
  "user_id": "user-789",
  "message": "Order processed successfully",
  "action": "order.process",
  "duration_ms": 145,
  "context": {
    "order_id": "order-456",
    "amount": 99.99,
    "status": "completed"
  }
}
```

## Part 3: Logging Implementation

### 3.1 Logging Best Practices

- Log at appropriate levels (don't over-log)
- Include context for debugging
- Never log sensitive data (passwords, tokens, PII)
- Use structured logging (JSON)
- Include request IDs for tracing
- Log both successes and failures
- Include timing information

### 3.2 Sensitive Data Protection

```python
def mask_sensitive_fields(data):
    """Mask sensitive fields before logging"""
    sensitive_fields = {
        'password', 'api_key', 'token', 'secret',
        'credit_card', 'ssn', 'email'
    }
    
    masked_data = data.copy()
    for field in sensitive_fields:
        if field in masked_data:
            masked_data[field] = '***REDACTED***'
    
    return masked_data

# Usage
logger.info('User authenticated', extra=mask_sensitive_fields({
    'user_id': 123,
    'email': 'user@example.com',
    'password': 'secret123'
}))
```

## Part 4: Error Response Format

### 4.1 Standard Error Response

```json
{
  "status": "error",
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request parameters",
    "request_id": "req-12345abc",
    "timestamp": "2024-01-03T05:15:00Z",
    "details": {
      "fields": [
        {
          "name": "email",
          "message": "Invalid email format"
        }
      ]
    }
  }
}
```

## Part 5: Distributed Tracing

### 5.1 Request ID Propagation

```python
import uuid
from contextvars import ContextVar

request_id_context = ContextVar('request_id', default=None)

def middleware_set_request_id(request, call_next):
    request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))
    request_id_context.set(request_id)
    response = call_next(request)
    response.headers['X-Request-ID'] = request_id
    return response

def get_request_id():
    return request_id_context.get()

# In logging
logger.info('Processing order', extra={
    'request_id': get_request_id(),
    'order_id': order_id
})
```

## Success Metrics

- 100% error handling coverage
- Zero unhandled exceptions in production
- <1% of errors due to unknown causes
- Complete audit trail maintained
- Log retention: 30 days minimum
- Search capability: Full-text search enabled
- MTTD for errors: <5 minutes
