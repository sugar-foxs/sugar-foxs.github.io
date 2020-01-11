---
layout:     post
title:      mybatis拦截器分页
subtitle:   
date:       2019-12-29
author:     sugar-foxs
catalog: 	true
tags:
    - mybatis
    - spring
---

本文主要介绍mybatis拦截器分页的实现。

<!-- more -->

- 分页有两种实现方式：内存分页，物理分页。
    - 内存分页，将所有数据存入内存再分页，这个做法在大数据量下性能很差，只适合在小数据量下使用；
    - 物理分页，通过在sql层面实现，在sql后面加上limit 1,5 的语句来实现，但是这样每个需要分页的sql都要加上这个语句。
- 所以最优的方法是使用拦截器拦截需要分页的sql实现物理分页，这样能够节省工作量，也能提高性能。本篇文章主要介绍mybatis的拦截器分页的实现，具体原理这里不讲。

# 拦截器
```
/**
 * @author guchunhui
 * 2019-12-29 13:51
 **/
 // 高版本得使用这个，否则会报错
@Intercepts({ @Signature(type = StatementHandler.class, method = "prepare", args = { Connection.class, Integer.class }) })
@Component
public class PageInterceptor implements Interceptor {
    Logger logger = LoggerFactory.getLogger(PageInterceptor.class);

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler)invocation.getTarget();
        MetaObject metaObject = MetaObject.forObject(statementHandler, SystemMetaObject.DEFAULT_OBJECT_FACTORY, SystemMetaObject.DEFAULT_OBJECT_WRAPPER_FACTORY,new DefaultReflectorFactory());
        MappedStatement mappedStatement = (MappedStatement) metaObject.getValue("delegate.mappedStatement");
        String id = mappedStatement.getId();

        //所有以ByPage结尾的dao层方法会被拦截
        if(id.matches(".+ByPage$")){

            BoundSql boundSql = statementHandler.getBoundSql();
            PaginationParam params = (PaginationParam)boundSql.getParameterObject();
            int currPage = params.getPageNum();
            int pageSize = params.getPageSize();

            String sql = boundSql.getSql();

            // 获取总数
            String countSql = "select count(*) from (" + sql + ")a";
            Connection connection = (Connection) invocation.getArgs()[0];
            PreparedStatement countStatement = connection.prepareStatement(countSql);
            ParameterHandler parameterHandler = (ParameterHandler) metaObject.getValue("delegate.parameterHandler");
            parameterHandler.setParameters(countStatement);
            ResultSet rs = countStatement.executeQuery();
            if(rs.next()){
                params.setTotalNum(rs.getInt(1));
            }

            // 生成分页语句
            String pageSql = sql + " limit " + (currPage-1) * pageSize + "," + pageSize;
            metaObject.setValue("delegate.boundSql.sql", pageSql);
        }
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object o) {
        return Plugin.wrap(o, this);
    }

    @Override
    public void setProperties(Properties properties) {}
}
```

PaginationParam负责分页参数和存储结果总数，你自己也可以算出总页数存入。
```
/**
 * @author guchunhui
 * 2019-12-29 14:42
 **/
@Data
public class PaginationParam extends BaseParam {
    private Integer pageNum;
    private Integer pageSize;
    private Integer totalNum;
}

```

> 使用分页拦截器需要在你之前使用mybatis操作数据库正常的情况下增加。
> dao层需要分页的方法只需要使用ByPage结尾就可以实现分页。
