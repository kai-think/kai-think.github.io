---
layout:     post
title:      SpringBoot全局异常管理
subtitle:   SpringBoot全局异常管理
date:       2024-06-25
author:     KAI
header-img: img/wallhaven-l8vp7y.jpg
catalog: true
tags:
    - SpringBoot
    - 异常处理
---
>SpringBoot全局异常管理
# SpringBoot全局异常管理

### 异常的处理逻辑

开发中不允许截留处理exception，应该转化为自定义的CustomException之后抛出去。我们只需要负责一件事，那就是捕获异常，并转化为自定义异常，最后抛出自定义异常信息。

### 异常抛出之后谁来处理

**ControllerAdvice**

监听所有的Controller，一旦controller抛出CustomException，就会在@ExceptionHandler(CustomException.class)注解下的方法里对该异常进行处理

### 流程

自定义异常类型

```java
public enum CustomExceptionType {

    OK(HttpStatus.OK.value(),"请求成功"),
    BAD_REQUEST(HttpStatus.BAD_REQUEST.value(),"您输入的数据错误或您没有权限访问资源！"),
    INTERNAL_SERVER_ERROR (HttpStatus.INTERNAL_SERVER_ERROR.value(),"系统出现异常，请您稍后再试或联系管理员！"),
    OTHER_ERROR(999,"系统出现未知异常，请联系管理员！");

    CustomExceptionType(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    private String desc;//异常类型中文描述

    private int code; //code

    public String getDesc() {
        return desc;
    }

    public int getCode() {
        return code;
    }
}
```

自定义异常继承运行时异常RuntimeException

```java
public class CustomException extends RuntimeException{
    private int code;
    private String message;

    private CustomException() {}

    public CustomException(CustomExceptionType exceptionType)
    {
        this.code = exceptionType.getCode();
        this.message = exceptionType.getDesc();
    }

    public CustomException(CustomExceptionType exceptionType, String message)
    {
        this.code = exceptionType.getCode();
        this.message = message;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    @Override
    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}
```

封装返回数据

```JAVA
package com.example.demo;

public class Result {

    private int code;
    private String message;
    private Object data;

    public Result() {
    }

    public static Result success(Object data) {
        return new Result(200, CustomExceptionType.OK.getDesc(), data);
    }

    public static Result success(String message, Object data) {
        return new Result(200, message, data);
    }

    public static Result success() {
        return new Result(200, CustomExceptionType.OK.getDesc(), null);
    }

    public static Result error(CustomExceptionType customExceptionType, String message) {
        return new Result(customExceptionType.getCode(), message, null);
    }

    public static Result error(CustomException customException) {
        return new Result(customException.getCode(), customException.getMessage(), null);
    }

    public Result(int code, String message, Object data) {
        this.code = code;
        this.message = message;
        this.data = data;
    }

    public int getCode() {
        return code;
    }

    public void setCode(int code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public Object getData() {
        return data;
    }

    public void setData(Object data) {
        this.data = data;
    }
}
```

全局异常处理器GlobalExceptionHandler

```JAVA
@ControllerAdvice
public class GlobalExceptionHandler {

    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);

    @ExceptionHandler(CustomException.class)
    @ResponseBody
    public Result handleCustomException(CustomException e) {
        if (e.getCode() == CustomExceptionType.INTERNAL_SERVER_ERROR.getCode()) {
            // 500异常做日志记录，比如发送邮件或者短信
            log.error("系统出现异常，异常信息：{}", e.getMessage());
        }
        return Result.error(e);
    }

    @ExceptionHandler(Exception.class)
    @ResponseBody
    public Result handleException(Exception e) {
        return Result.error(CustomExceptionType.OTHER_ERROR, e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseBody
    public Result handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        return Result.error(CustomExceptionType.BAD_REQUEST, e.getBindingResult().getFieldError().getDefaultMessage());
    }

    @ExceptionHandler(BindException.class)
    @ResponseBody
    public Result handleBindException(BindException e) {
        return Result.error(CustomExceptionType.BAD_REQUEST, e.getBindingResult().getFieldError().getDefaultMessage());
    }

    @ExceptionHandler(IllegalArgumentException.class)
    @ResponseBody
    public Result handleIllegalArgumentException(IllegalArgumentException e) {
        return Result.error(
                new CustomException(CustomExceptionType.BAD_REQUEST, e.getMessage())
        );
    }


}
```

