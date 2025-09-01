在高并发系统中，接口限流是保障服务稳定的重要手段。本文介绍一种基于 **IP 地址和方法维度** 的限流实现方案，使用 Spring MVC 拦截器结合 Guava RateLimiter 和 Caffeine 缓存，实现高效且可热更新的限流策略。

整体设计思路是：每个用户通过 IP 进行区分，不同接口访问频率不同，因此限流粒度需要精确到方法级别。对于每个 IP + 方法组合，使用 Guava RateLimiter 控制访问速率，并通过 Caffeine 缓存存储限流器，支持自动过期和容量限制，同时可在运行时动态更新速率。

在实现上，首先定义一个 Spring MVC 拦截器 `RateLimitInterceptor`，在 `preHandle` 方法中获取请求的 IP 地址和方法签名，并将两者组合作为限流 Key。通过 `RateLimiterService` 获取对应的 RateLimiter，并尝试获取令牌，如果获取失败，则直接返回 HTTP 429 响应和 JSON 错误信息。这样可以保证每个用户对每个接口独立限流，并且超出限制的请求不会进入业务逻辑层。

```Java
@Component
@Slf4j
public class RateLimitInterceptor implements HandlerInterceptor {
    @Resource
    private RateLimiterService rateLimiterService;

    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (!(handler instanceof HandlerMethod handlerMethod)) return true;

        String ip = getIp(request);
        Method method = handlerMethod.getMethod();
        String key = ip + ":" + method.getDeclaringClass().getName() + "." + method.getName();

        if (!rateLimiterService.getRateLimiter(key).tryAcquire()) {
            response.setStatus(429);
            response.setContentType("application/json");
            response.setCharacterEncoding("UTF-8");
            response.getWriter().write(new ObjectMapper().writeValueAsString(
                    ResultUtils.error(ErrorCode.TOO_MANY_REQUEST, "请求频率过高，请稍后再试。")
            ));
            log.debug("key :{} 请求频率过高！", key);
            return false;
        }
        return true;
    }

    private String getIp(HttpServletRequest request) {
        String ip = request.getRemoteAddr();
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !"unknown".equalsIgnoreCase(xForwardedFor)) {
            ip = xForwardedFor.split(",")[0].trim();
        }
        return ip;
    }
}

```

`RateLimiterService` 负责管理缓存中的 RateLimiter。Caffeine 缓存用于存储每个 IP + 方法对应的 RateLimiter，支持 `expireAfterAccess` 自动过期和 `maximumSize` 容量限制，避免内存膨胀。同时提供方法动态更新全局默认速率，缓存中已有的限流器也会同步调整。

```Java
@Service
public class RateLimiterService {
    private Cache<String, RateLimiter> limiterCache;

    @Value("${app.rateLimiter.defaultPermitsPerSecond}")
    private volatile double defaultPermitsPerSecond;

    @Value("${app.rateLimiter.expireAfterAccess}")
    private volatile int expireAfterAccess;

    @Value("${app.rateLimiter.maximumSize}")
    private volatile int maximumSize;

    @PostConstruct
    public void init() {
        limiterCache = Caffeine.newBuilder()
                .expireAfterAccess(expireAfterAccess, TimeUnit.SECONDS)
                .maximumSize(maximumSize)
                .build();
    }

    public RateLimiter getRateLimiter(String key) {
        return limiterCache.get(key, k -> RateLimiter.create(defaultPermitsPerSecond));
    }

    public synchronized void updateDefaultPermitsPerSecond(double newRate) {
        this.defaultPermitsPerSecond = newRate;
        limiterCache.asMap().values().forEach(l -> l.setRate(newRate));
    }
}

```

在 Web 配置中，将拦截器注册到所有请求路径上，实现全局限流。同时配置 CORS 允许跨域请求携带 Cookie：

```Java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Resource
    private RateLimitInterceptor rateLimitInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(rateLimitInterceptor).addPathPatterns("/**");
    }

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowCredentials(true)
                .allowedOriginPatterns("*")
                .allowedMethods("GET", "POST", "PUT", "DELETE", "OPTIONS")
                .allowedHeaders("*")
                .exposedHeaders("*");
    }
}

```

这种方案优点在于精确控制每个用户对每个接口的访问速率，使用 Guava RateLimiter 平滑速率，Caffeine 提供高性能缓存，且可动态调整速率，无需重启服务。需要注意的是，当前方案为本地 JVM 限流，多实例部署时需结合 Redis 或 Redisson 实现分布式限流，同时 IP 获取和缓存策略可以根据实际业务进一步优化。