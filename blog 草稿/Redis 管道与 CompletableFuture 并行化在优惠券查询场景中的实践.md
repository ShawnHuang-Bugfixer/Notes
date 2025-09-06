在电商业务中，**优惠券查询**是一类高频场景。用户在下单时，系统需要快速返回其「可用优惠券」和「不可用优惠券」，并按照优惠金额排序，以便前端展示最优选择。

查询用户拥有哪些优惠券只需请求一次 Redis 即可拿到用户拥有的优惠券 Id 集合。显然如果循环遍历 id 集合请求 Redis 十分耗时，一个规模为 n 的 id 集合会产生额外 n - 1 次 RTT ，应该使用批处理获取详细优惠券模板信息，一次 RTT 内就可以获取详细的优惠券信息集合。

优惠券按照使用范围分为通用优惠券和指定商品优惠券，计算优惠券折扣金额是需要分别处理。同时优惠券还有立减券、折扣券、满减券三种类型，有三种不同的折扣计算策略。通用优惠券直接根据折扣策略将优惠券加入到可用 / 不可用优惠券列表，加入可用优惠券列表前计算折扣金额即可，而指定商品优惠券还需要判断是否适用于用户当前请求中携带的商品。

每张优惠券都需要解析 JSON 规则，并根据订单金额、商品范围计算可用性和优惠力度。对于 CPU 密集的逻辑，如果串行执行，耗时过长。可以适用并行计算优化。

## 技术挑战

在实现过程中，主要面临两方面的性能瓶颈：
1. **Redis I/O 消耗**：  
    用户可能持有数百张优惠券，逐条查询 Redis 将带来大量网络往返。
2. **CPU 计算开销**：  
    每张优惠券都需要解析 JSON 规则，并根据订单金额、商品范围计算可用性和优惠力度。对于 CPU 密集的逻辑，如果串行执行，耗时过长。

## 技术方案

为解决上述问题，我们采用了 **Redis 管道 + CompletableFuture 并行化** 的组合方案。

### 1. Redis 管道批量查询

首先，用户领券列表存储在 Redis 的有序集合中：

```Java
Set<String> rangeUserCoupons = stringRedisTemplate.opsForZSet()
        .range(String.format(USER_COUPON_TEMPLATE_LIST_KEY, userId), 0, -1);
```

根据领券记录拼接出优惠券模板的 Redis Key，然后通过 **Pipeline** 一次性批量获取优惠券详情：

```Java
List<Object> rawCouponDataList = stringRedisTemplate.executePipelined((RedisCallback<Object>) connection -> {
    redisCouponTemplateKeys.forEach(each -> connection.hashCommands().hGetAll(each.getBytes()));
    return null;
});

```

这样避免了每张券都单独请求 Redis 的 N 次网络往返，大幅降低了 IO 开销。

### 2. 按使用范围分类

获取到优惠券模板后，按照 `goods` 字段是否为空，分为两类：

- `goods` 为空：全店铺通用优惠券。
    
- `goods` 非空：仅指定商品可用。

```Java
Map<Boolean, List<CouponTemplateQueryRespDTO>> partitioned =
        JSON.parseArray(JSON.toJSONString(rawCouponDataList), CouponTemplateQueryRespDTO.class)
            .stream()
            .collect(Collectors.partitioningBy(coupon -> StrUtil.isEmpty(coupon.getGoods())));

```

### 3. CompletableFuture 异步并行计算

由于优惠券规则计算是 CPU 密集型操作，我们使用线程池配合 **CompletableFuture** 并行执行。

- **全店铺券**：直接按订单总金额判断可用性。
    
- **指定商品券**：只对指定商品金额进行判断。
    

```Java
CompletableFuture<Void> emptyGoodsTasks = CompletableFuture.allOf(
    goodsEmptyList.stream().map(each -> CompletableFuture.runAsync(() -> {
        handleCouponLogic(each, requestParam.getOrderAmount(), availableCouponList, notAvailableCouponList);
    }, executorService)).toArray(CompletableFuture[]::new)
);

CompletableFuture<Void> notEmptyGoodsTasks = CompletableFuture.allOf(
    goodsNotEmptyList.stream().map(each -> CompletableFuture.runAsync(() -> {
        BigDecimal goodsAmount = goodsRequestMap.get(each.getGoods()).getGoodsAmount();
        handleCouponLogic(each, goodsAmount, availableCouponList, notAvailableCouponList);
    }, executorService)).toArray(CompletableFuture[]::new)
);

CompletableFuture.allOf(emptyGoodsTasks, notEmptyGoodsTasks).thenRun(() -> {
    availableCouponList.sort((c1, c2) -> c2.getCouponAmount().compareTo(c1.getCouponAmount()));
}).join();

```

### 4. 优惠券规则计算逻辑

在计算优惠券可用性时，按类型分别处理：

- **立减券**：直接设置优惠金额。
    
- **满减券**：若订单金额 ≥ 使用门槛，则生效。
    
- **折扣券**：若订单金额 ≥ 使用门槛，则计算折扣金额并取最大优惠限制。

```Java
switch (coupon.getType()) {
    case 0: // 立减
        coupon.setCouponAmount(maximumDiscountAmount);
        availableList.add(coupon);
        break;
    case 1: // 满减
        if (orderAmount.compareTo(termsOfUse) >= 0) {
            coupon.setCouponAmount(maximumDiscountAmount);
            availableList.add(coupon);
        } else {
            notAvailableList.add(coupon);
        }
        break;
    case 2: // 折扣
        if (orderAmount.compareTo(termsOfUse) >= 0) {
            BigDecimal discountAmount = orderAmount.multiply(discountRate)
                                                   .min(maximumDiscountAmount);
            coupon.setCouponAmount(discountAmount);
            availableList.add(coupon);
        } else {
            notAvailableList.add(coupon);
        }
        break;
    default:
        throw new ClientException("无效的优惠券类型");
}

```

```Java
/**  
 * 查询用户可用优惠券列表接口  
 */  
@Service  
@RequiredArgsConstructor  
public class CouponQueryServiceImpl implements CouponQueryService {  
    private final StringRedisTemplate stringRedisTemplate;  
  
    // 在我们本次的业务场景中，属于是 CPU 密集型任务，设置 CPU 的核心数即可  
    private final ExecutorService executorService = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(), Runtime.getRuntime().availableProcessors(), 9999, TimeUnit.SECONDS, new SynchronousQueue<>(), new ThreadPoolExecutor.CallerRunsPolicy());  
  
    @Override  
    public QueryCouponsRespDTO listQueryUserCoupons(QueryCouponsReqDTO requestParam) {  
        // 1. 获取 Redis 中的用户优惠券列表  
        Set<String> rangeUserCoupons = stringRedisTemplate.opsForZSet().range(String.format(USER_COUPON_TEMPLATE_LIST_KEY, UserContext.getUserId()), 0, -1);  
  
        if (rangeUserCoupons == null || rangeUserCoupons.isEmpty()) {  
            return QueryCouponsRespDTO.builder().availableCouponList(new ArrayList<>()).notAvailableCouponList(new ArrayList<>()).build();  
        }  
  
        // 构建 couponTemplate Redis Key 并添加到集合  
        List<String> redisCouponTemplateKeys = rangeUserCoupons.stream().map(each -> StrUtil.split(each, "_").get(0)).map(each -> String.format(COUPON_TEMPLATE_KEY, each)).toList();  
  
        // 管道批量执行 Redis 指令获取缓存中的 couponTemplate        List<Object> rawCouponDataList = stringRedisTemplate.executePipelined((RedisCallback<Object>) connection -> {  
            redisCouponTemplateKeys.forEach(each -> connection.hashCommands().hGetAll(each.getBytes()));  
            return null;  
        });  
  
        // 2. 分别处理指定商品和未执行商品的优惠券。解析 Redis 数据，并按 `goods` 字段进行分区处理  
        Map<Boolean, List<CouponTemplateQueryRespDTO>> partitioned = JSON.parseArray(JSON.toJSONString(rawCouponDataList), CouponTemplateQueryRespDTO.class).stream().collect(Collectors.partitioningBy(coupon -> StrUtil.isEmpty(coupon.getGoods())));  
  
        // 拆分后的两个列表  
        List<CouponTemplateQueryRespDTO> goodsEmptyList = partitioned.get(true); // goods 为空的列表  
        List<CouponTemplateQueryRespDTO> goodsNotEmptyList = partitioned.get(false); // goods 不为空的列表  
  
        // 针对当前订单可用/不可用的优惠券列表  
        List<QueryCouponsDetailRespDTO> availableCouponList = Collections.synchronizedList(new ArrayList<>());  
        List<QueryCouponsDetailRespDTO> notAvailableCouponList = Collections.synchronizedList(new ArrayList<>());  
  
        // Step 2: 并行处理 goodsEmptyList 和 goodsNotEmptyList 中的每个元素  
        CompletableFuture<Void> emptyGoodsTasks = CompletableFuture.allOf(goodsEmptyList.stream().map(each -> CompletableFuture.runAsync(() -> {  
            QueryCouponsDetailRespDTO resultCouponDetail = BeanUtil.toBean(each, QueryCouponsDetailRespDTO.class);  
            JSONObject jsonObject = JSON.parseObject(each.getConsumeRule());  
            handleCouponLogic(resultCouponDetail, jsonObject, requestParam.getOrderAmount(), availableCouponList, notAvailableCouponList);  
        }, executorService)).toArray(CompletableFuture[]::new));  
  
        Map<String, QueryCouponGoodsReqDTO> goodsRequestMap = requestParam.getGoodsList().stream().collect(Collectors.toMap(QueryCouponGoodsReqDTO::getGoodsNumber, Function.identity()));  
        CompletableFuture<Void> notEmptyGoodsTasks = CompletableFuture.allOf(goodsNotEmptyList.stream().map(each -> CompletableFuture.runAsync(() -> {  
            QueryCouponsDetailRespDTO resultCouponDetail = BeanUtil.toBean(each, QueryCouponsDetailRespDTO.class);  
            QueryCouponGoodsReqDTO couponGoods = goodsRequestMap.get(each.getGoods());  
            if (couponGoods == null) {  
                notAvailableCouponList.add(resultCouponDetail);  
            } else {  
                JSONObject jsonObject = JSON.parseObject(each.getConsumeRule());  
                handleCouponLogic(resultCouponDetail, jsonObject, couponGoods.getGoodsAmount(), availableCouponList, notAvailableCouponList);  
            }  
        }, executorService)).toArray(CompletableFuture[]::new));  
  
        // Step 3: 等待两个异步任务集合完成  
        CompletableFuture.allOf(emptyGoodsTasks, notEmptyGoodsTasks).thenRun(() -> {  
            // 与业内标准一致，按最终优惠力度从大到小排序  
            availableCouponList.sort((c1, c2) -> c2.getCouponAmount().compareTo(c1.getCouponAmount()));  
        }).join();  
  
        // 构建最终结果并返回  
        return QueryCouponsRespDTO.builder().availableCouponList(availableCouponList).notAvailableCouponList(notAvailableCouponList).build();  
    }  
  
    /**  
     * 根据优惠券类型和消费规则，将优惠券添加到优惠券可用/不可用集合。  
     * @param resultCouponDetail 优惠券  
     * @param consumeRule 优惠券消费规则  
     * @param rawOrderAmount 订单金额  
     * @param availableCouponList 可用优惠券集合  
     * @param notAvailableCouponList 不可用优惠券集合  
     */  
    private void handleCouponLogic(QueryCouponsDetailRespDTO resultCouponDetail, JSONObject consumeRule, BigDecimal rawOrderAmount, List<QueryCouponsDetailRespDTO> availableCouponList, List<QueryCouponsDetailRespDTO> notAvailableCouponList) {  
        BigDecimal termsOfUse = consumeRule.getBigDecimal("termsOfUse");  
        BigDecimal maximumDiscountAmount = consumeRule.getBigDecimal("maximumDiscountAmount");  
  
        switch (resultCouponDetail.getType()) {  
            case 0: // 立减券  
                resultCouponDetail.setCouponAmount(maximumDiscountAmount);  
                availableCouponList.add(resultCouponDetail);  
                break;  
            case 1: // 满减券  
                if (rawOrderAmount.compareTo(termsOfUse) >= 0) {  
                    resultCouponDetail.setCouponAmount(maximumDiscountAmount);  
                    availableCouponList.add(resultCouponDetail);  
                } else {  
                    notAvailableCouponList.add(resultCouponDetail);  
                }  
                break;  
            case 2: // 折扣券  
                if (rawOrderAmount.compareTo(termsOfUse) >= 0) {  
                    BigDecimal discountRate = consumeRule.getBigDecimal("discountRate");  
                    BigDecimal multiply = rawOrderAmount.multiply(discountRate);  
                    if (multiply.compareTo(maximumDiscountAmount) >= 0) {  
                        resultCouponDetail.setCouponAmount(maximumDiscountAmount);  
                    } else {  
                        resultCouponDetail.setCouponAmount(multiply);  
                    }  
                    availableCouponList.add(resultCouponDetail);  
                } else {  
                    notAvailableCouponList.add(resultCouponDetail);  
                }  
                break;  
            default:  
                throw new ClientException("无效的优惠券类型");  
        }  
    }  
  
    /**  
     * 单线程版本，好理解一些。上面的多线程就是基于这个版本演进的  
     */  
    public QueryCouponsRespDTO listQueryUserCouponsBySync(QueryCouponsReqDTO requestParam) {  
        // 1. 查询 Redis 用户优惠券列表、提取 couponTemplateId 并执行管道命令获取 couponTemplate 详细信息集合。  
        Set<String> rangeUserCoupons = stringRedisTemplate.opsForZSet().range(String.format(USER_COUPON_TEMPLATE_LIST_KEY, UserContext.getUserId()), 0, -1);  
        if (CollUtil.isEmpty(rangeUserCoupons)) {  
            return QueryCouponsRespDTO.builder().availableCouponList(new ArrayList<>()).notAvailableCouponList(new ArrayList<>()).build();  
        }  
        List<String> redisCouponTemplateKeys = rangeUserCoupons.stream().map(each -> StrUtil.split(each, "_").get(0)).map(each -> String.format(COUPON_TEMPLATE_KEY, each)).toList();  
        List<Object> couponTemplateList = stringRedisTemplate.executePipelined((RedisCallback<String>) connection -> {  
            redisCouponTemplateKeys.forEach(each -> connection.hashCommands().hGetAll(each.getBytes()));  
            return null;  
        });  
  
        List<CouponTemplateQueryRespDTO> rawCouponTemplateDTOList = JSON.parseArray(JSON.toJSONString(couponTemplateList), CouponTemplateQueryRespDTO.class);  
        Map<Boolean, List<CouponTemplateQueryRespDTO>> partitioned = rawCouponTemplateDTOList.stream().collect(Collectors.partitioningBy(coupon -> StrUtil.isEmpty(coupon.getGoods())));  
  
        // 2. 分别处理不同优惠券模板类型  
        List<CouponTemplateQueryRespDTO> goodsEmptyList = partitioned.get(true); // goods 为空的列表  
        List<CouponTemplateQueryRespDTO> goodsNotEmptyList = partitioned.get(false); // goods 不为空的列表  
  
        // 针对当前订单可用/不可用的优惠券列表  
        List<QueryCouponsDetailRespDTO> availableCouponList = new ArrayList<>();  
        List<QueryCouponsDetailRespDTO> notAvailableCouponList = new ArrayList<>();  
  
        goodsEmptyList.forEach(each -> {  
            JSONObject jsonObject = JSON.parseObject(each.getConsumeRule());  
            QueryCouponsDetailRespDTO resultQueryCouponDetail = BeanUtil.toBean(each, QueryCouponsDetailRespDTO.class);  
            BigDecimal maximumDiscountAmount = jsonObject.getBigDecimal("maximumDiscountAmount");  
            switch (each.getType()) {  
                case 0: // 立减券  
                    resultQueryCouponDetail.setCouponAmount(maximumDiscountAmount);  
                    availableCouponList.add(resultQueryCouponDetail);  
                    break;  
                case 1: // 满减券  
                    // orderAmount 大于或等于 termsOfUse                    if (requestParam.getOrderAmount().compareTo(jsonObject.getBigDecimal("termsOfUse")) >= 0) {  
                        resultQueryCouponDetail.setCouponAmount(maximumDiscountAmount);  
                        availableCouponList.add(resultQueryCouponDetail);  
                    } else {  
                        notAvailableCouponList.add(resultQueryCouponDetail);  
                    }  
                    break;  
                case 2: // 折扣券  
                    // orderAmount 大于或等于 termsOfUse                    if (requestParam.getOrderAmount().compareTo(jsonObject.getBigDecimal("termsOfUse")) >= 0) {  
                        BigDecimal multiply = requestParam.getOrderAmount().multiply(jsonObject.getBigDecimal("discountRate"));  
                        if (multiply.compareTo(maximumDiscountAmount) >= 0) {  
                            resultQueryCouponDetail.setCouponAmount(maximumDiscountAmount);  
                        } else {  
                            resultQueryCouponDetail.setCouponAmount(multiply);  
                        }  
                        availableCouponList.add(resultQueryCouponDetail);  
                    } else {  
                        notAvailableCouponList.add(resultQueryCouponDetail);  
                    }  
                    break;  
                default:  
                    throw new ClientException("无效的优惠券类型");  
            }  
        });  
  
        Map<String, QueryCouponGoodsReqDTO> goodsRequestMap = requestParam.getGoodsList().stream().collect(Collectors.toMap(QueryCouponGoodsReqDTO::getGoodsNumber, Function.identity(), (existing, replacement) -> existing));  
  
        goodsNotEmptyList.forEach(each -> {  
            QueryCouponGoodsReqDTO couponGoods = goodsRequestMap.get(each.getGoods());  
            if (couponGoods == null) {  
                notAvailableCouponList.add(BeanUtil.toBean(each, QueryCouponsDetailRespDTO.class));  
            } else {  
                JSONObject jsonObject = JSON.parseObject(each.getConsumeRule());  
                QueryCouponsDetailRespDTO resultQueryCouponDetail = BeanUtil.toBean(each, QueryCouponsDetailRespDTO.class);  
                switch (each.getType()) {  
                    case 0: // 立减券  
                        resultQueryCouponDetail.setCouponAmount(jsonObject.getBigDecimal("maximumDiscountAmount"));  
                        availableCouponList.add(resultQueryCouponDetail);  
                        break;  
                    case 1: // 满减券  
                        // goodsAmount 大于或等于 termsOfUse                        if (couponGoods.getGoodsAmount().compareTo(jsonObject.getBigDecimal("termsOfUse")) >= 0) {  
                            resultQueryCouponDetail.setCouponAmount(jsonObject.getBigDecimal("maximumDiscountAmount"));  
                            availableCouponList.add(resultQueryCouponDetail);  
                        } else {  
                            notAvailableCouponList.add(resultQueryCouponDetail);  
                        }  
                        break;  
                    case 2: // 折扣券  
                        // goodsAmount 大于或等于 termsOfUse                        if (couponGoods.getGoodsAmount().compareTo(jsonObject.getBigDecimal("termsOfUse")) >= 0) {  
                            BigDecimal discountRate = jsonObject.getBigDecimal("discountRate");  
                            resultQueryCouponDetail.setCouponAmount(couponGoods.getGoodsAmount().multiply(discountRate));  
                            availableCouponList.add(resultQueryCouponDetail);  
                        } else {  
                            notAvailableCouponList.add(resultQueryCouponDetail);  
                        }  
                        break;  
                    default:  
                        throw new ClientException("无效的优惠券类型");  
                }  
            }  
        });  
  
        // 与业内标准一致，按最终优惠力度从大到小排序  
        availableCouponList.sort((c1, c2) -> c2.getCouponAmount().compareTo(c1.getCouponAmount()));  
  
        return QueryCouponsRespDTO.builder().availableCouponList(availableCouponList).notAvailableCouponList(notAvailableCouponList).build();  
    }  
}
```