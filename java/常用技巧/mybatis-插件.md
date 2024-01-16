
mybatis 在执行具体的 sql 之前可以做一些操作，比如记录特定日志等等

基本配置

```xml
<bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
      <property name="failFast" value="true" />
      <property name="dataSource" ref="dataSource" />
      <property name="mapperLocations" value="classpath*:daoMapper/*.xml" />
      <property name="plugins">
         <list>
            <bean class="ParameterValidateInterceptor">
               <property name="include" value="WxworkUserInfoDao,WxworkPartyInfoDao"/>
            </bean>
         </list>
      </property>
   </bean>
```

这里可以配置只针对特定的 Dao 类。

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.atomic.AtomicBoolean;

import org.apache.commons.lang3.StringUtils;
import org.apache.ibatis.binding.BindingException;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.mapping.SqlSource;
import org.apache.ibatis.plugin.Interceptor;
import org.apache.ibatis.plugin.Intercepts;
import org.apache.ibatis.plugin.Invocation;
import org.apache.ibatis.plugin.Plugin;
import org.apache.ibatis.plugin.Signature;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

@Intercepts({
    @Signature(type = Executor.class, method = "query", args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}),
    @Signature(type = Executor.class, method = "queryCursor", args = {MappedStatement.class, Object.class, RowBounds.class}),
})
public class ParameterValidateInterceptor implements Interceptor {

    private static final String ID_KEY = "id";
    private static final String CORP_ID_KEY = "corpId";
    private static final Map<String, String> includeMap = new HashMap<>();
    private static final Map<String, String> excludeMap = new HashMap<>();

    private String include;
    private String exclude;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        boolean checkResult = processArgs(args);
        if (!checkResult) {
            throw new BindingException("Parameter 'corpId' not found.");
        }
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    private void initProperties(String property, Map<String, String> map) {
        String ids = property;
        if (StringUtils.isNotBlank(ids)) {
            String[] idArray = ids.split(",");
            for (int i = 0; i < idArray.length; i++) {
                map.put(idArray[i], idArray[i]);
            }
        }
    }

    private boolean processArgs(Object[] args) {

        if (args == null || args.length == 0) {
            return true;
        }

        MappedStatement mappedStatement = (MappedStatement) args[0];
        String id = mappedStatement.getId();
        // 在排除范围
        if (excludeMap.get(id) != null) {
            return true;
        }

        // 不在处理范围
        String clzzName = getId(mappedStatement);
        if (includeMap.get(clzzName) == null) {
            return true;
        }

        SqlSource sqlSource = mappedStatement.getSqlSource();
        BoundSql boundSql = sqlSource.getBoundSql(args[1]);
        //缺少参数
        if (boundSql.getParameterMappings() == null || boundSql.getParameterMappings().isEmpty()) {
            return false;
        }

        AtomicBoolean result = new AtomicBoolean(false);
        boundSql.getParameterMappings().forEach(p -> {
            if (ID_KEY.equals(p.getProperty()) || CORP_ID_KEY.equals(p.getProperty())) {
                result.set(true);
            }
        });
        return result.get();
    }

    private String getId(MappedStatement mappedStatement) {
        String id = mappedStatement.getId();
        // 这里id带有方法
        int inx = id.lastIndexOf(".");
        return id.substring(0, inx);
    }

    public String getInclude() {
        return include;
    }

    public void setInclude(String include) {
        this.include = include;
        initProperties(include, includeMap);
    }

    public String getExclude() {
        return exclude;
    }

    public void setExclude(String exclude) {
        this.exclude = exclude;
        initProperties(exclude, excludeMap);
    }
}
```