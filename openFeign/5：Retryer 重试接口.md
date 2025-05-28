1：简述

    重试的触发点，在 3：方法调用 SynchronousMethodHandler 里面触发 ，只要捕获到 RetryableException 异常，则会触发：重试
    每个 HTTP 请求执行时会克隆出一个新的 Retryer 实例，用于跟踪当前请求的重试状态。
    
    假如允许重试：第一次重试默认等待：100ms，后续每次都是前一次乘以：1.5 指数递增，但是受最大允许值控制（默认：1秒），默认最多重试：5次
    
2：类定义

    public interface Retryer extends Cloneable {
    }
    
3：类成员变量
    
        private final int maxAttempts;
        private final long period;
        private final long maxPeriod;
        int attempt;
        long sleptForMillis;
    
4：核心方法：

    // 为每个http请求 clone一个实例
    @Override
    public Retryer clone() {
      return new Default(period, maxPeriod, maxAttempts);
    }
    
    
    // 默认重试策略：初始等待时间：100ms，最大等待时间：1 秒，最多重试 5 次（总共最多执行 6 次）
    public Default() {
      this(100, SECONDS.toMillis(1), 5);
    }

    
    
    // 重试方法
    public void continueOrPropagate(RetryableException e) {
          if (attempt++ >= maxAttempts) {
            throw e;
          }
          // 下次重试时间间隔
          long interval;
          // 优先获取：retryAfter
          if (e.retryAfter() != null) {
            interval = e.retryAfter().getTime() - currentTimeMillis();
            if (interval > maxPeriod) {
              interval = maxPeriod;
            }
            if (interval < 0) {
              return;
            }
          } else {
            // retryAfter 为空，则根据 等待市场乘以：1.5倍，最大：1s
            interval = nextMaxInterval();
          }
          try {
            Thread.sleep(interval);
          } catch (InterruptedException ignored) {
            Thread.currentThread().interrupt();
            throw e;
          }
          sleptForMillis += interval;
    }
    
    
    // 每次重试等待时间指数增长，乘以 1.5 倍。例如：第一次：100，第二次：150，第三次：225，第四次：337.5 第五次：506
    long nextMaxInterval() {
      long interval = (long) (period * Math.pow(1.5, attempt - 1));
      return interval > maxPeriod ? maxPeriod : interval;
    }


    
    