# 标准 RBAC 权限模型
标准 RBAC (Role-Based Access Control，基于角色的方案控制)，用户不直接拥有权限，而是通过绑定角色获取角色对应的权限映射。

标准的 RBAC 模型基于角色间接获取权限，一般涉及用户表、角色表、权限表、用户角色映射表和角色权限映射表五张表:

| 表名                         | 功能                      |
| -------------------------- | ----------------------- |
| `users`（用户表）               | 存储系统用户信息                |
| `roles`（角色表）               | 定义系统中的角色，如管理员、普通用户      |
| `permissions`（权限表）         | 定义可操作的权限，如“查看订单”、“编辑用户” |
| `user_role`（用户角色映射表）       | 关联用户与角色，多对多关系           |
| `role_permission`（角色权限映射表） | 关联角色与权限，多对多关系           |
权限获取流程：
1. 查询用户角色列表
2. 根据角色查询角色权限列表
3. 根据权限列表判断是否有权限

# 动态 RBAC 模型

图库系统包含有公共、私人和团队三类空间，用户在不同空间内对图片有不同的权限。按照标准的 RBAC 模型，需要构建用户角色映射表，在该系统中，用户在不同空间有不同角色，如果强行基于标准 RBAC 模型则需要构建用户在不同空间内用户角色映射表，会极大复杂化用户角色映射表，并导致及其复杂的 SQL，因此不宜采用标准 RBAC 模型做法。

通过上述分析，现有系统构建静态用户角色映射表及其困难，因此需要动态计算用户在不同空间内的角色，而角色和权限的映射关系固定，只要计算出用户在不同空间内的角色，就可以根据角色权限映射表获取用户的权限集合。

现在关键点在于如何动态计算出用户在不同空间中的角色，显而易见，用户的角色在不同空间中有不同的权限，因此计算角色时涉及到 spaceId、userId、spaceUserId 和 pictureId。直接将这些 Id 封装到用户角色上下文 UserAuthContext 中，并从请求 RequestContextHolder 上下文容器中拿到 HttpServletRequest request 请求，解析请求得到相关信息封装为 UserAuthContext 对象，接下来就是业务计算过程。根据用户权限上下文计算用户角色。

计算角色就是以空间维度划分，分别讨论用户在不同空间下拥有的角色，在通过该角色得出用户在具体空间内的权限。每次尝试获取权限都需要进行计算，一种优化方式就是缓存用户在指定空间内的权限。

 将计算过程整合到 Sa-Token 框架中，并利用 Sa-Token 工具类即可实现鉴权。

```Java
public class StpInterfaceImpl implements StpInterface {  
    @Value("${server.servlet.context-path}")  
    private String contextPath;  
  
    @Resource  
    @Lazy    private UserService userService;  
  
    @Resource  
    private PictureService pictureService;  
  
    @Resource  
    private SpaceService spaceService;  
  
    @Resource  
    private SpaceUserService spaceUserService;  
  
    /**  
     * 返回一个账号所拥有的权限码集合  
     */  
    @Override  
    public List<String> getPermissionList(Object loginId, String loginType) {  
        UserAuthContext userAuthContext = getUserAuthContextByCurrentRequest();  
        Long userId = Long.valueOf(loginId.toString());  
        Long pictureId = userAuthContext.getPictureId();  
        Long spaceId = userAuthContext.getSpaceId();  
        Long spaceUserId = userAuthContext.getSpaceUserId();  
        return getPermissions(spaceId, userId, pictureId, spaceUserId);  
    }  
  
    public void addPermissionsBySpaceUser(ArrayList<String> permissions, SpaceUser dbSpaceuser) {  
        if (ObjUtil.isNotNull(dbSpaceuser)) {  
            switch (dbSpaceuser.getSpaceRole()) {  
                case "admin":  
                    log.debug("团队空间拥有者" + "空间id：{}", dbSpaceuser.getSpaceId());  
                    permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.TEAM_SPACE_OWNER.getValue()));  
                    break;  
                case "editor":  
                    log.debug("团队空间编辑者" + "空间id：{}", dbSpaceuser.getSpaceId());  
                    permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.TEAM_SPACE_EDITOR.getValue()));  
                    break;  
                case "viewer":  
                    log.debug("团队空间浏览者" + "空间id：{}", dbSpaceuser.getSpaceId());  
                    permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.TEAM_SPACE_VIEWER.getValue()));  
                    break;  
                default:  
                    break;  
            }  
        }  
    }  
  
    public List<String> getPermissions(Long spaceId, @NonNull Long userId, Long pictureId, Long spaceUserId) {  
        ArrayList<String> permissions = new ArrayList<>();  
        // 1. userId 在 User 表中 userRole 为 admin，系统管理员  
        User dbUser = userService.getById(userId);  
        if (dbUser.getUserRole().equals(RoleEnum.SYSTEM_ADMIN.getValue())) {  
            log.debug("系统管理员：{}", userId);  
            permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.SYSTEM_ADMIN.getValue()));  
        }  
        // 2. pictureId 在 picture 表中有记录  
        if (ObjUtil.isNotNull(pictureId)) {  
            Picture dbPicture = pictureService.getById(pictureId);  
            if (ObjUtil.isNotNull(dbPicture)) {  
                Long dbPictureSpaceId = dbPicture.getSpaceId();  
                if (ObjUtil.isNull(dbPictureSpaceId) && dbPicture.getUserId().equals(userId)) {  
                    // 2.1 如果图片属于公有空间，即 spaceId == null，且 pictureId 属于 userId，公共图片拥有者  
                    permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.PUBLIC_SPACE_PICTURE_OWNER.getValue()));  
                } else {  
                    // 2.2 如果图片不属于公有空间，即 spaceId ！= null  
                    Space dbSpace = spaceService.getById(dbPictureSpaceId);  
                    if (ObjUtil.isNotNull(dbSpace)) {  
                        Integer spaceType = dbSpace.getSpaceType();  
                        if (spaceType.equals(0)) {  
                            // 2.2.1 空间类型为私人空间  
                            //       spaceId 属于 userId，私人空间拥有者  
                            if (dbSpace.getUserId().equals(userId)) {  
                                log.debug("私人空间拥有者" + "空间id: {}", dbPictureSpaceId);  
                                permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.PRIVATE_SPACE_OWNER.getValue()));  
                            }  
                        }  
                        if (spaceType.equals(1)) {  
                            // 2.2.2 空间类型为团队空间  
                            //      查询 space_user 表，根据 spaceId 和 userId 查询 spaceRole 有记录，赋予对应角色  
                            QueryWrapper<SpaceUser> spaceUserQueryWrapper = new QueryWrapper<>();  
                            spaceUserQueryWrapper.select("spaceRole")  
                                    .eq("spaceId", dbPictureSpaceId)  
                                    .eq("userId", userId);  
                            SpaceUser dbSpaceUser = spaceUserService.getOne(spaceUserQueryWrapper);  
                            if (ObjUtil.isNotNull(dbSpaceUser)) {  
                                addPermissionsBySpaceUser(permissions, dbSpaceUser);  
                            }  
                        }  
                    }  
                }  
            }  
        }  
        // 3. spaceId 在 space 表中有记录  
        if (ObjUtil.isNotNull(spaceId)) {  
            Space dbSpace = spaceService.getById(spaceId);  
            if (ObjUtil.isNotNull(dbSpace)) {  
                // 3.1 spaceId 对应的空间类型 spaceType == 0 为私人空间  
                // 3.1.1 spaceId 属于 userId，私人空间拥有者  
                if (dbSpace.getSpaceType().equals(0) && dbSpace.getUserId().equals(userId)) {  
                    log.debug("私人空间拥有者" + "空间id: {}", spaceId);  
                    permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.PRIVATE_SPACE_OWNER.getValue()));  
                }  
                // 3.2 spaceId 对应的空间类型 spaceType == 1 为团队空间  
                // 3.2.1 根据 spaceId 和 userId 查询 space_user 表 spaceRole 有记录  
                //       * spaceRole == admin 团队空间拥有者  
                //       * spaceRole == viewer 团队空间浏览者  
                //       * spaceRole == editor 团队空间编辑者  
                if (dbSpace.getSpaceType().equals(1)) {  
                    QueryWrapper<SpaceUser> spaceUserQueryWrapper = new QueryWrapper<>();  
                    spaceUserQueryWrapper.select("spaceRole")  
                            .eq("spaceId", spaceId)  
                            .eq("userId", userId);  
                    SpaceUser dbSpaceuser = spaceUserService.getOne(spaceUserQueryWrapper);  
                    addPermissionsBySpaceUser(permissions, dbSpaceuser);  
                }  
            }  
        }  
        // 4. spaceUserid 在 space_user 表中有记录  
        //       * spaceRole == admin 团队空间拥有者  
        //       * spaceRole == viewer 团队空间浏览者  
        //       * spaceRole == editor 团队空间编辑者  
        if (ObjUtil.isNotNull(spaceUserId)) {  
            SpaceUser dbSpaceuser = spaceUserService.getById(spaceUserId);  
            Long teamSpaceId = dbSpaceuser.getSpaceId();  
            QueryWrapper<SpaceUser> spaceUserQueryWrapper = new QueryWrapper<>();  
            spaceUserQueryWrapper.select("spaceRole")  
                    .eq("spaceId", teamSpaceId)  
                    .eq("userId", userId);  
            dbSpaceuser = spaceUserService.getOne(spaceUserQueryWrapper);  
            addPermissionsBySpaceUser(permissions, dbSpaceuser);  
        }  
        permissions.addAll(AuthManager.getPermissionsByRole(RoleEnum.PUBLIC_SPACE_OWNER.getValue()));  
        log.debug("权限列表：{}", permissions);  
        return permissions;  
    }  
  
  
    /**  
     * 返回一个账号所拥有的角色标识集合 (权限与角色可分开校验)  
     */    @Override  
    public List<String> getRoleList(Object loginId, String loginType) {  
        // 本 list 仅做模拟，实际项目中要根据具体业务逻辑来查询角色  
        List<String> list = new ArrayList<>();  
        list.add("admin");  
        list.add("super-admin");  
        return list;  
    }  
  
    private UserAuthContext getUserAuthContextByCurrentRequest() {  
        HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();  
        String contentType = request.getContentType();  
        UserAuthContext userAuthContext;  
        // 处理 get 请求和 post 请求  
        if (ContentType.JSON.getValue().equals(contentType)) { // 处理 post 请求  
            String body = ServletUtil.getBody(request);  
            userAuthContext = JSONUtil.toBean(body, UserAuthContext.class);  
        } else {  
            // 处理 get 请求  
            Map<String, String> paramMap = ServletUtil.getParamMap(request);  
            userAuthContext = BeanUtil.toBean(paramMap, UserAuthContext.class);  
        }  
        Long id = userAuthContext.getId();  
        if (ObjUtil.isNotNull(id)) {  
            // uri: /api/picture/**  
            String requestUri = request.getRequestURI();  
            String partUri = requestUri.replace(contextPath + "/", "");  
            String moduleName = StrUtil.subBefore(partUri, "/", false);  
            switch (moduleName) {  
                case "picture":  
                    userAuthContext.setPictureId(id);  
                    break;  
                case "spaceUser":  
                    userAuthContext.setSpaceUserId(id);  
                    break;  
                case "space":  
                    userAuthContext.setSpaceId(id);  
                    break;  
                default:  
            }  
        }  
        return userAuthContext;  
    }  
}
```

```Java
StpUtil.checkPermissionOr(PermissionConstants.PUBLIC_MODIFY_IMAGE);
```
 
```Java
/**
* 从请求上下文容器中获取当前请求并解析请求封装为用户权限上下文。
*/
private UserAuthContext getUserAuthContextByCurrentRequest() {  
    HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();  
    String contentType = request.getContentType();  
    UserAuthContext userAuthContext;  
    // 处理 get 请求和 post 请求  
    if (ContentType.JSON.getValue().equals(contentType)) { // 处理 post 请求  
        String body = ServletUtil.getBody(request);  
        userAuthContext = JSONUtil.toBean(body, UserAuthContext.class);  
    } else {  
        // 处理 get 请求  
        Map<String, String> paramMap = ServletUtil.getParamMap(request);  
        userAuthContext = BeanUtil.toBean(paramMap, UserAuthContext.class);  
    }  
    Long id = userAuthContext.getId();  
    if (ObjUtil.isNotNull(id)) {  
        // uri: /api/picture/**  
        String requestUri = request.getRequestURI();  
        String partUri = requestUri.replace(contextPath + "/", "");  
        String moduleName = StrUtil.subBefore(partUri, "/", false);  
        switch (moduleName) {  
            case "picture":  
                userAuthContext.setPictureId(id);  
                break;  
            case "spaceUser":  
                userAuthContext.setSpaceUserId(id);  
                break;  
            case "space":  
                userAuthContext.setSpaceId(id);  
                break;  
            default:  
        }  
    }  
    return userAuthContext;  
}
```

## 包装 Reqest 让请求体可重复读取
默认情况下，`HttpServletRequest.getInputStream()`/`getReader()` 返回的是对**底层 TCP socket / 请求流**，只能顺序读取一次，第二次无法读取数据。但是从请求中解析用户权限上下文需要重复读取请求体，因此需要对请求进行包装。

利用过滤器对 HttpServletReqeust 进行包装：

```Java
@Order(1)  
@Component  
public class HttpRequestWrapperFilter implements Filter {  
  
    @Override  
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws ServletException, IOException {  
        if (request instanceof HttpServletRequest) {  
            HttpServletRequest servletRequest = (HttpServletRequest) request;  
            String contentType = servletRequest.getHeader(Header.CONTENT_TYPE.getValue());  
            if (ContentType.JSON.getValue().equals(contentType)) {  
                // 可以再细粒度一些，只有需要进行空间权限校验的接口才需要包一层  
                chain.doFilter(new RequestWrapper(servletRequest), response);  
            } else {  
                chain.doFilter(request, response);  
            }  
        }  
    }  
  
}
```

流本质上是一次性的：一旦数据被读出，指针就移动到末尾，除非手动重置，否则无法再次读取。请求包装类在第一次读取时把整个请求体缓存一份存到 body 中，并重写 getReader() 和 getInputStream() 方法，是两个方法针对缓存的 body 进行读取，可以重复读取。
```Java
@Slf4j  
public class RequestWrapper extends HttpServletRequestWrapper {  
  
    private final String body;  
  
    public RequestWrapper(HttpServletRequest request) {  
        super(request);  
        StringBuilder stringBuilder = new StringBuilder();  
        try (InputStream inputStream = request.getInputStream(); BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(inputStream))) {  
            char[] charBuffer = new char[128];  
            int bytesRead = -1;  
            while ((bytesRead = bufferedReader.read(charBuffer)) > 0) {  
                stringBuilder.append(charBuffer, 0, bytesRead);  
            }  
        } catch (IOException ignored) {  
        }  
        body = stringBuilder.toString();  
    }  
  
    @Override  
    public ServletInputStream getInputStream() throws IOException {  
        final ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(body.getBytes());  
        return new ServletInputStream() {  
            @Override  
            public boolean isFinished() {  
                return false;  
            }  
  
            @Override  
            public boolean isReady() {  
                return false;  
            }  
  
            @Override  
            public void setReadListener(ReadListener readListener) {  
            }  
  
            @Override  
            public int read() throws IOException {  
                return byteArrayInputStream.read();  
            }  
        };  
  
    }  
  
    @Override  
    public BufferedReader getReader() throws IOException {  
        return new BufferedReader(new InputStreamReader(this.getInputStream()));  
    }  
  
    public String getBody() {  
        return this.body;  
    }  
  
}
```