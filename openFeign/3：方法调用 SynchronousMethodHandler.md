一：简述：
    
    调用feign接口，最终都会到这个方法里面来：这里做了：组装：Resttemplate，发送请求，解析响应，失败重试等
    具体步骤：
        invoke() 方法被调用，生成请求模板并设置重试机制；
        通过 executeAndDecode() 发出 HTTP 请求；
        请求成功：
        如果返回类型是 Response，原样返回；
        如果是 2xx 或 404（可配置），则用解码器转换为方法返回值；
        请求失败：
        抛出异常，重试器决定是否重试；
        超过重试次数则抛出最终异常；
        关闭响应体以释放连接。

二：类定义：
    
    class SynchronousMethodHandler implements MethodHandler{
    }
    
三：核心方法：
    
    // 方法调用入口，每当调用一个 Feign 接口方法（比如 userClient.getUserById(1L)），最终都会走到这里。
    public Object invoke(Object[] argv) throws Throwable {
      RequestTemplate template = buildTemplateFromArgs.create(argv); // 构建请求模板
      Options options = findOptions(argv); // 获取请求配置（超时等）
      Retryer retryer = this.retryer.clone(); // 克隆重试器（保证线程安全）
    
      while (true) {
        try {
          return executeAndDecode(template, options); // 执行请求并解析返回值
        } catch (RetryableException e) { // 如果是可重试的异常
          try {
            retryer.continueOrPropagate(e); // 判断是否继续重试
          } catch (RetryableException th) { // 不再重试，抛出
            Throwable cause = th.getCause();
            if (propagationPolicy == UNWRAP && cause != null) {
              throw cause; // 解包异常，抛出原始异常
            } else {
              throw th; // 否则抛出 RetryableException
            }
          }
    
          // 打印重试日志
          if (logLevel != Logger.Level.NONE) {
            logger.logRetry(metadata.configKey(), logLevel);
          }
    
          continue; // 继续重试
        }
      }
    }
    
    // 真正的远程调用、响应处理和解码。
    Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
      Request request = targetRequest(template); // 构建 HTTP 请求
    
      // 日志记录请求
      if (logLevel != Logger.Level.NONE) {
        logger.logRequest(metadata.configKey(), logLevel, request);
      }
    
      Response response;
      long start = System.nanoTime();
      try {
        response = client.execute(request, options); // 发送请求
      } catch (IOException e) {
        // 记录 IO 异常
        if (logLevel != Logger.Level.NONE) {
          logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime(start));
        }
        throw errorExecuting(request, e);
      }
    
      long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
      boolean shouldClose = true; // 是否需要关闭 response.body()
    
      try {
        // 打印响应日志 + 包装流
        if (logLevel != Logger.Level.NONE) {
          response = logger.logAndRebufferResponse(metadata.configKey(), logLevel, response, elapsedTime);
        }
    
        // 特殊返回类型：如果方法返回的是 Response
        if (Response.class == metadata.returnType()) {
          if (response.body() == null) return response;
    
          if (response.body().length() == null || response.body().length() > MAX_RESPONSE_BUFFER_SIZE) {
            shouldClose = false;
            return response;
          }
    
          // 否则读取 body 流为字节数组（避免连接未释放）
          byte[] bodyData = Util.toByteArray(response.body().asInputStream());
          return response.toBuilder().body(bodyData).build();
        }
    
        // 正常 2xx 响应
        if (response.status() >= 200 && response.status() < 300) {
          if (void.class == metadata.returnType()) {
            return null;
          } else {
            Object result = decode(response); // 解码为实际返回值
            shouldClose = closeAfterDecode;
            return result;
          }
        }
    
        // 特殊处理：404 + decode404 = true 且返回值不是 void
        else if (decode404 && response.status() == 404 && void.class != metadata.returnType()) {
          Object result = decode(response);
          shouldClose = closeAfterDecode;
          return result;
        }
    
        // 其他非 2xx 状态：交由 errorDecoder 处理（抛出异常）
        else {
          throw errorDecoder.decode(metadata.configKey(), response);
        }
    
      } catch (IOException e) {
        if (logLevel != Logger.Level.NONE) {
          logger.logIOException(metadata.configKey(), logLevel, e, elapsedTime);
        }
        throw errorReading(request, response, e); // 读取时出错
      } finally {
        // 根据 shouldClose 判断是否需要关闭响应体流
        if (shouldClose) {
          ensureClosed(response.body());
        }
      }
    }

