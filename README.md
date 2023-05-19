# v10-Interface

### 由于是个老项目,使用的jdk版本是1.6,使用框架Struts,无法像Spring Cloud 一样引入熔断，降级等组件,所以利用线程池做依赖外部接口调用快速熔断

### 统一调用接口,封装请求,简化调用方式

### V10ApiEnum.java
```java
package com.fxt.util.http;

import com.fxt.util.http.response.CompanySwitchDTO;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;

import java.util.Map;

/**
 * 〈一句话功能简述〉<br/>
 * 配置请求
 *
 * @author shihao.liu
 * @see [相关类/方法]（可选）
 * @since [产品/模块版本] （可选）
 */
public enum V10ApiEnum {

    /**
     * 公司灰度开关
     */
    COMPANY_SWITCH_URL("/api/systoolbff/getCompanySwitch", HttpMethod.POST,
            Map.class, CompanySwitchDTO.class, 2000L, "http.apollo.gateway.url"),

    /**
     * 获取灰度公司端口数
     */
    GET_COMPANY_PORT_URL("/http/api2/getCompany?companyUuid={companyUuid}", HttpMethod.GET, Map.class, String.class,
            2000L, "mall.url.58"),

    /**
     * 获取配置中心
     */
    GET_CONFIG_CENTER_URL("/api/commonbff/metadata/listCompanySetByKeys", HttpMethod.POST, Map.class, String.class,
            3000L, "http.apollo.gateway.url"),

    /**
     * 开启公司 app端用
     */
    OPEN_COMPANY("/api/commonservice/company/openCompany", HttpMethod.POST, Map.class, String.class,
            3000L, "http.apollo.gateway.url"),

    /**
     * 关闭公司 app端用
     */
    STOP_COMPANY("/api/commonservice/company/stopCompany", HttpMethod.POST, Map.class, String.class,
            3000L, "http.apollo.gateway.url"),

    /**
     * 踢出apollo所有在线用户
     */
    LOGOUT_ALL("/api/homebff/homeUser/logoutAll", HttpMethod.POST, Map.class, String.class,
            3000L, "http.apollo.gateway.url"),

    /**
     * 根据类型code查询字典名称
     */
    GET_DICTIONARY_INFO_MAP("/api/commonservice/dictionaryInfo/getDictionaryInfoMap", HttpMethod.POST, Map.class, String.class,
            3000L, "http.apollo.gateway.url");

    V10ApiEnum(String uri, HttpMethod httpMethod, Class<?> requestType, Class<?> responseType, Long timeOut, String hostConfigName) {
        this.uri = uri;
        this.httpMethod = httpMethod;
        this.requestType = requestType;
        this.responseType = responseType;
        this.timeOut = timeOut;
        this.hostConfigName = hostConfigName;
    }

    public <RQ, RS> ResponseEntity<RS> send(RQ requestBody, HttpHeaders httpHeaders, Object... uriVariables) {
        return V10RestTemplate.send(this, httpHeaders, (Class<RQ>) requestType, (Class<RS>) responseType, requestBody, uriVariables);
    }

    /**
     * URL
     */
    private String uri;

    /**
     * 请求方式
     */
    private HttpMethod httpMethod;

    /**
     * 请求类型
     */
    private Class<?> requestType;

    /**
     * 返回类型
     */
    private Class<?> responseType;

    /**
     * 超时时间
     */
    private Long timeOut;

    /**
     * host配置文件
     */
    private String hostConfigName;


    public String getUri() {
        return uri;
    }

    public HttpMethod getHttpMethod() {
        return httpMethod;
    }

    public Class<?> getRequestType() {
        return requestType;
    }

    public Class<?> getResponseType() {
        return responseType;
    }

    public Long getTimeOut() {
        return timeOut;
    }

    public String getHostConfigName() {
        return hostConfigName;
    }
}
```

### V10RestTemplate.java
```java
package com.fxt.util.http;

import com.alibaba.fastjson.JSONObject;
import com.fxt.core.base.exception.FxtBusinessException;
import com.fxt.core.util.FxtWebProperties;
import com.fxt.core.util.SpringUtil;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;

/**
 * 〈一句话功能简述〉<br/>
 * V10RestTemplate 对接外部系统
 * 可以为每个请求设置超时时间
 *
 * @author shihao.liu
 * @see [相关类/方法]（可选）
 * @since [产品/模块版本] （可选）
 */
public class V10RestTemplate {


    protected static final transient Log log = LogFactory.getLog(V10RestTemplate.class);

    public static <RQ, RS> ResponseEntity<RS> send(V10ApiEnum api, HttpHeaders httpHeaders,
                                                   Class<RQ> requestType, Class<RS> responseType,
                                                   RQ requestBody, Object... uriVariables) {
        if (!requestType.isInstance(requestBody)) {
            throw new FxtBusinessException("请求参数错误");
        }
        String host = FxtWebProperties.getProperties().getProperty(api.getHostConfigName());
        log.info("V10RestTemplate send url:" + (host + api.getUri()) + ",param:" + JSONObject.toJSONString(requestBody));
        return RestTemplateWrapper.execute(host + api.getUri(), api.getHttpMethod(),
                httpHeaders, responseType, JSONObject.toJSONString(requestBody), api.getTimeOut(), uriVariables);
    }

}
```

### RestTemplateWrapper.java 
```java
package com.fxt.util.http;

import com.alibaba.fastjson.JSONObject;
import com.fxt.util.LoginUserUtil;
import com.fxt.util.sync.ThreadPoolUtil;
import org.apache.commons.logging.Log;
import org.apache.commons.logging.LogFactory;
import org.springframework.http.*;
import org.springframework.http.converter.StringHttpMessageConverter;
import org.springframework.web.client.RestTemplate;

import java.nio.charset.StandardCharsets;
import java.util.concurrent.Callable;
import java.util.concurrent.FutureTask;
import java.util.concurrent.TimeUnit;

/**
 * 〈一句话功能简述〉<br/>
 * RestTemplate包装类
 *
 * @author shihao.liu
 * @see [相关类/方法]（可选）
 * @since [产品/模块版本] （可选）
 */
public class RestTemplateWrapper {

    protected static final transient Log log = LogFactory.getLog(RestTemplateWrapper.class);


    static final RestTemplate REST_TEMPLATE = new RestTemplate();

    static {
        REST_TEMPLATE.getMessageConverters().set(1, new StringHttpMessageConverter(StandardCharsets.UTF_8));
    }


    /**
     * 封装http请求
     *
     * @param url          请求地址
     * @param httpMethod   请求方式
     * @param httpHeaders  header
     * @param responseType 返回类型
     * @param params       请求参数
     * @param timeOut      超时时间 毫秒
     * @param <T>          返回类型
     * @param uriVariables uri中的可变参数
     * @return ResponseEntity<T>
     */
    public static <T> ResponseEntity<T> execute(final String url, final HttpMethod httpMethod, final HttpHeaders httpHeaders,
                                                final Class<T> responseType,
                                                final String params,
                                                Long timeOut, final Object... uriVariables) {
        final String companyUuid = LoginUserUtil.getCompanyUuid();
        try {
            // 创建异步任务
            FutureTask<ResponseEntity<T>> futureTask = new FutureTask<>(new Callable<ResponseEntity<T>>() {
                @Override
                public ResponseEntity<T> call() {
                    HttpEntity<String> httpEntity = new HttpEntity<>(params, httpHeaders);
                    return REST_TEMPLATE.exchange(url, httpMethod, httpEntity, responseType, uriVariables);
                }
            });
            // 提交线程池执行
            ThreadPoolUtil.getThreadPool().submit(futureTask);
            // 等待执行结果
            ResponseEntity<T> responseEntity = futureTask.get(timeOut, TimeUnit.MILLISECONDS);
            // 发送日志的
            if (responseEntity.getStatusCode() == HttpStatus.OK) {
                // 发送日志到日志平台
                V10RequestLogHelper.send(url, params, JSONObject.toJSONString(responseEntity.getBody()));
            }
            return responseEntity;
        } catch (Exception e) {
            log.error(e.getMessage(), e);
            // 从日志平台获取数据返回
            return V10RequestLogHelper.getHttpResponse(url, params, responseType, companyUuid);
        }
    }
}
```