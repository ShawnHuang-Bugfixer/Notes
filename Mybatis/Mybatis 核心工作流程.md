
# Mybatis 核心工作流程

![[attachments/Pasted image 20250527191212.png]]
# `Mybatis` 二级缓存机制


![[attachments/Pasted image 20250527191354.png]]
![[attachments/Pasted image 20250527191407.png]]

# Mybatis 接口代理

在 mapper.xml 文件中定义的 sql 语句会以 Key - Value 格式被加载到 MappedStatement 对象，其中 key 为 namespace + id，并存储在 Configuration 对象中。

sqlsession 可以直接通过 key 执行 sql 语句，但存在硬编码问题，官方推荐使用接口代理调用 sql。

```XMl
<mapper namespace="com.example.mapper.UserMapper">
  <select id="selectUserById" resultType="User" parameterType="int">
    SELECT * FROM user WHERE id = #{id}
  </select>
</mapper>
```

```Java
public interface UserMapper {
    User selectUserById(int id);
}

SqlSession session = sqlSessionFactory.openSession();
UserMapper mapper = session.getMapper(UserMapper.class); // 推荐！
User user = mapper.selectUserById(1);

SqlSession session = sqlSessionFactory.openSession();
User user = session.selectOne("com.example.mapper.UserMapper.selectUserById", 1); // 不推荐！
```

# 延迟加载

MyBatis 使用 **CGLIB 或 Javassist 动态代理** 技术，给延迟加载的属性生成代理对象，只有在调用 getter 方法时，才触发数据库查询。

```XML
<settings>
  <!-- 开启全局延迟加载 -->
  <setting name="lazyLoadingEnabled" value="true"/>
  
  <!-- 是否按需加载属性（true：按需，false：加载一个属性就加载全部关联属性）-->
  <setting name="aggressiveLazyLoading" value="false"/>
</settings>
```

只有 user 对象调用 getter 相关方法时，才会执行 sql 获取数据。
```XML
<resultMap id="orderMap" type="Order">
  <id property="id" column="id"/>
  <result property="orderNumber" column="order_number"/>
  <association property="user" column="user_id" javaType="User" select="selectUserById" fetchType="lazy"/> 
</resultMap>

```