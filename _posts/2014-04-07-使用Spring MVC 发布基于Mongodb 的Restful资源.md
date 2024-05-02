使用Spring MVC发布Restful资源不难，读写Mongodb也不难，但是能不能让两者无缝地对接起来呢？也就是说，让Spring MVC来做retrieve/stringify的工作，我们的方法就像这样：

```java
public DBObject createModel(@RequestBody DBObject model)
```

按照Spring文档的说法，当一个@RequestMapping的方法没有@ResponesBody时，Spring会尝试渲染View；如果方法同时也带了@ResponseBody，Spring会查找当前已经注册的HttpMessageConverter,通过这些Converter得到希望的结果。所以，我们要做的第一步就是实现DBObject的Converter：

```java
package com.narcissu5.util;

import com.mongodb.BasicDBObject;
import com.mongodb.DBObject;
import com.mongodb.util.JSON;
import org.springframework.http.HttpInputMessage;
import org.springframework.http.HttpOutputMessage;
import org.springframework.http.MediaType;
import org.springframework.http.converter.HttpMessageConverter;
import org.springframework.http.converter.HttpMessageNotReadableException;
import org.springframework.http.converter.HttpMessageNotWritableException;

import java.io.*;
import java.util.ArrayList;
import java.util.List;

/**
 * Created by Narcissu5 on 2014/4/5.
 */
public class DBObjectMessageConverter implements HttpMessageConverter<DBObject> {

    @Override
    public boolean canRead(Class<?> aClass, MediaType mediaType) {
        return DBObject.class.isAssignableFrom(aClass);
    }

    @Override
    public boolean canWrite(Class<?> aClass, MediaType mediaType) {
        return DBObject.class.isAssignableFrom(aClass);
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        List<MediaType> supports = new ArrayList<MediaType>();
        supports.add(MediaType.APPLICATION_JSON);
        return supports;
    }

    @Override
    public DBObject read(Class<? extends DBObject> aClass, HttpInputMessage httpInputMessage) throws IOException, HttpMessageNotReadableException {
        Object object = JSON.parse(readToEnd(httpInputMessage.getBody()));
        if (object == null) {
            return new BasicDBObject();
        } else {
            return (DBObject) object;
        }
    }

    @Override
    public void write(DBObject dbObject, MediaType mediaType, HttpOutputMessage httpOutputMessage) throws IOException, HttpMessageNotWritableException {
        httpOutputMessage.getBody().write(dbObject.toString().getBytes());
    }

    protected String readToEnd(InputStream is) throws IOException {
        java.util.Scanner s = new java.util.Scanner(is).useDelimiter("\\A");
        return s.hasNext() ? s.next() : "";
    }
}
```

然后需要需要将我们的Converter注册到Spring的Context中去，官方的注册方法是这样：

```xml
<bean class="org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter">
    <property name="messageConverters">
      <util:list id="beanList">
        <ref bean="stringHttpMessageConverter"/>
        <ref bean="marshallingHttpMessageConverter"/>
      </util:list>
    </property
</bean>

<bean id="stringHttpMessageConverter"
       class="org.springframework.http.converter.StringHttpMessageConverter"/>

<bean id="marshallingHttpMessageConverter"
      class="org.springframework.http.converter.xml.MarshallingHttpMessageConverter">
  <property name="marshaller" ref="castorMarshaller" />
  <property name="unmarshaller" ref="castorMarshaller" />
</bean>

<bean id="castorMarshaller" class="org.springframework.oxm.castor.CastorMarshaller"/>
```

但是。。。。不起作用，还是Stackoverflow靠谱，实际上应该这样注册：

```xml
<mvc:annotation-driven>
        <mvc:message-converters register-defaults="true">
            <bean class="com.narcissu5.util.DBObjectMessageConverter"/>
        </mvc:message-converters>
    </mvc:annotation-driven>
```

register-defaults表示同时也注册默认的Converter，spring内置了几种Converter：

- `ByteArrayHttpMessageConverter` converts byte arrays.
    
- `StringHttpMessageConverter` converts strings.
    
- `FormHttpMessageConverter` converts form data to/from a MultiValueMap<String, String>.
    
- `SourceHttpMessageConverter` converts to/from a javax.xml.transform.Source.
    

然后是Controller，因为有Converter的帮助，我们不用再费心处理json转换的工作了：

```java
package com.narcissu5.mvc;

import com.mongodb.*;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

import javax.servlet.http.HttpServletResponse;
import java.net.UnknownHostException;

/**
 * Created by Narcissu5 on 2014/4/6.
 */
@Controller
@RequestMapping(value = "/models")
public class ModelController {
    @ResponseBody
    @RequestMapping(method = RequestMethod.POST)
    public DBObject createModel(@RequestBody DBObject model) throws UnknownHostException {
        MongoClient client = new MongoClient("localhost");
        DBCollection colModels = client.getDB("MyDB").getCollection("models");
        colModels.insert(model);
        return model;
    }

    @ResponseBody
    @RequestMapping(method = RequestMethod.PUT)
    public DBObject updateModel(@RequestBody DBObject model) throws UnknownHostException {
        MongoClient client = new MongoClient("localhost");
        DBCollection colModels = client.getDB("MyDB").getCollection("models");
        colModels.save(model);
        return model;
    }

    @ResponseBody
    @RequestMapping(method = RequestMethod.GET)
    public DBObject queryModel() throws UnknownHostException {
        MongoClient client = new MongoClient("localhost");
        DBCollection colModels = client.getDB("MyDB").getCollection("models");
        DBCursor cursor = colModels.find();
        BasicDBList all = new BasicDBList();
        try {
            while (cursor.hasNext()) {
                all.add(cursor.next());
            }
        } finally {
            cursor.close();
        }
        return all;
    }

    @ResponseBody
    @RequestMapping(method = RequestMethod.PATCH)
    public DBObject getModel(@RequestBody DBObject query) throws UnknownHostException {
        MongoClient client = new MongoClient("localhost");
        DBCollection colModels = client.getDB("MyDB").getCollection("models");
        return colModels.findOne(query);
    }

    @RequestMapping(method = RequestMethod.DELETE)
    public void deleteModel(@RequestBody DBObject std, HttpServletResponse resp) throws UnknownHostException {
        MongoClient client = new MongoClient("localhost");
        DBCollection colModels = client.getDB("MyDB").getCollection("models");
        WriteResult result = colModels.remove(std);
        if (result.getN() > 0) {
            resp.setStatus(HttpServletResponse.SC_OK);
        } else {
            resp.setStatus(HttpServletResponse.SC_NO_CONTENT);
        }
    }
}
```

注意这里的查找和删除不像一般的Restful使用/models/{id}这样的路径，这是因为在mongodb中。默认的id是对象而非整型，而且官方也不鼓励使用别的主键类型，因此我们通过消息体将包含主键的对象提交上来，相应的查找的方法也从GET换成了PATCH。

最后是测试，使用Powershell：

```javascript
function Assert($expression,$message){
    Write-Host $message -NoNewline
    if($expression){
        Write-Host "`tSuccess" -ForegroundColor Green
    } else {
        Write-Host "`tFail" -ForegroundColor Red
    }
}

$url = "http://localhost:8080/models"
#POST /models
$exception = $false
$body = ConvertTo-Json @{name=[GUID]::NewGuid().ToString()}
try{
    $resp = Invoke-WebRequest -Body $body -ContentType "application/json" -Method Post -Uri $url #-Headers $headers
} catch {
    $exception = $true
}
Assert (-not $exception -and ($resp.StatusCode -eq 200)) "POST /models" 

#PUT models
$exception = $false
$ret = ConvertFrom-Json ([System.Text.Encoding]::UTF8.GetString($resp.Content))
$ret.name = [DateTime]::Now.ToString()
$body = ConvertTo-Json $ret 
try{
    $resp = Invoke-WebRequest -Body $body -ContentType "application/json" -Method PUT -Uri $url 
} catch {
    $exception = $true
}
Assert (-not $exception -and ($resp.StatusCode -eq 200)) "PUT /models"

#GET /models
$exception = $false
try{
    $resp = Invoke-WebRequest -Method GET -Uri $url 
} catch {
    $exception = $true
}
$ret = ConvertFrom-Json ([System.Text.Encoding]::UTF8.GetString($resp.Content))
Assert (-not $exception -and ($ret.Count -gt 0) -and ($resp.StatusCode -eq 200)) "GET /models"

#GET /models
$exception = $false
$body = ConvertTo-Json (@{_id=$ret[0]._id})
try{
    $resp = Invoke-WebRequest -Body $body -Method PATCH -Uri $url 
} catch {
    $exception = $true
}
Assert  (-not $exception -and ($resp.StatusCode -eq 200)) "PATCH /models"

#DELETE /models
$exception = $false
try{
    $resp = Invoke-WebRequest -Body $body -Method DELETE -Uri $url 
} catch {
    $exception = $true
}
Assert  (-not $exception -and ($resp.StatusCode -eq 200)) "DELETE /models"
```

打完收功=W=