在 Java Web 开发中，`HttpServletRequest` 的请求体（RequestBody）默认只能被读取一次。一旦通过 `getInputStream()` 或 `getReader()` 读取后，流会被关闭，后续无法再读取。这在日志打印、参数校验、请求签名校验等需要多次读取请求体的场景下，会带来不便。

# 为什么 RequestBody 默认只能读取一次？
`HttpServletRequest` 的请求体是基于输入流的（`ServletInputStream`）。流的本质是**只能顺序向前读取**的数据通道，一旦数据被消费（读到末尾），后续无法回退或重新读取。如果某个过滤器、拦截器或 Controller 提前读取了请求体，后续代码将拿不到数据。

既然无法重复读取流，那么就直接将请求体数据缓存到内存，供后续读取，缓存可以重复读取。

# 缓存请求体
利用高优先级过滤器在请求到达接口前直接将请求体数据缓存到内存中，通过继承 `HttpServletRequestWrapper`，重写 `getInputStream()` 和 `getReader()`，让它们返回基于缓存的流，而不是原始流。这样，后续无论多少次读取，都从缓存中读取数据。

```Java
import javax.servlet.ReadListener;
import javax.servlet.ServletInputStream;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletRequestWrapper;
import java.io.*;

public class CachedBodyHttpServletRequest extends HttpServletRequestWrapper {

    private final byte[] body;

    public CachedBodyHttpServletRequest(HttpServletRequest request) throws IOException {
        super(request);
        // 读取原始请求体并缓存
        body = getBodyBytes(request);
    }

    private byte[] getBodyBytes(HttpServletRequest request) throws IOException {
        try (InputStream inputStream = request.getInputStream();
             ByteArrayOutputStream baos = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[1024];
            int len;
            while ((len = inputStream.read(buffer)) != -1) {
                baos.write(buffer, 0, len);
            }
            return baos.toByteArray();
        }
    }

    @Override
    public ServletInputStream getInputStream() {
        final ByteArrayInputStream bais = new ByteArrayInputStream(body);
        return new ServletInputStream() {
            @Override
            public int read() {
                return bais.read();
            }

            @Override
            public boolean isFinished() {
                return bais.available() == 0;
            }

            @Override
            public boolean isReady() {
                return true;
            }

            @Override
            public void setReadListener(ReadListener readListener) {
                // 无需实现
            }
        };
    }

    @Override
    public BufferedReader getReader() {
        return new BufferedReader(new InputStreamReader(this.getInputStream()));
    }

    public String getBodyString() {
        return new String(body);
    }
}
```

```Java
import javax.servlet.*;
import javax.servlet.http.HttpServletRequest;
import java.io.IOException;

public class CachedBodyFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        CachedBodyHttpServletRequest cachedRequest = new CachedBodyHttpServletRequest(httpRequest);

        // 继续传递包装后的 request
        chain.doFilter(cachedRequest, response);
    }
}

```

