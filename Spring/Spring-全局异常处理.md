---
date: 2025-04-23
aliases:
  - 全局异常处理
---
Spring 使用注解 `@ControllerAdvice` `@ExceptionHandler` 注解声明全局异常处理。利用 `AOP` 对指定的 Controller 织入异常处理逻辑。

```Java
@ControllerAdvice
@ResponseBody
public class GlobalExceptionHandler {

    @ExceptionHandler(BaseException.class)
    public ResponseEntity<?> handleAppException(BaseException ex, HttpServletRequest request) {
      //......
    }

    @ExceptionHandler(value = ResourceNotFoundException.class)
    public ResponseEntity<ErrorReponse> handleResourceNotFoundException(ResourceNotFoundException ex, HttpServletRequest request) {
      //......
    }
}
```