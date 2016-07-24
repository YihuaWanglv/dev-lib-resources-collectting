


1. How to I write a unit test for a Spring Boot Controller endpoint

An example test for your controller can be something as simple as
```
public class DemoApplicationTests {

    final String BASE_URL = "http://localhost:8080/";
    private MockMvc mockMvc;

    @Before
    public void setup() {
        this.mockMvc = standaloneSetup(new HelloWorld()).build();
    }

    @Test
    public void testSayHelloWorld() throws Exception {
        this.mockMvc.perform(get("/").accept(MediaType.parseMediaType("application/json;charset=UTF-8")))
                .andExpect(status().isOk())
                .andExpect(content().contentType("application/json"));

    }
}
```

The new testing improvements that debuted in Spring Boot 1.4.M2 can help reduce the amount of code you need to write situation such as these.

The test would look like so:
```
@RunWith(SpringRunner.class)
@WebMvcTest(HelloWorld.class)
public class UserVehicleControllerTests {

    @Autowired
    private MockMvc mvc;

    @Test
    public void testSayHelloWorld() throws Exception {
        this.mockMvc.perform(get("/").accept(MediaType.parseMediaType("application/json;charset=UTF-8")))
                .andExpect(status().isOk())
                .andExpect(content().contentType("application/json"));

    }

}
```

还有一种方式是使用TestRestTemplate

```
package controller;

import java.util.Date;
import java.util.HashMap;
import java.util.Map;

import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.SpringApplicationConfiguration;
import org.springframework.boot.test.TestRestTemplate;
import org.springframework.boot.test.WebIntegrationTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.web.client.RestTemplate;

import ipicture.service.post.AppServicePost;
import ipicture.service.post.model.JsonObject;
import ipicture.service.post.model.Subject;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(classes = AppServicePost.class)
@WebIntegrationTest("server.port:8083")// 使用0表示端口号随机，也可以具体指定如8888这样的固定端口
public class SubjectControllerTest {

    private RestTemplate template = new TestRestTemplate();
    @Value("${server.port}")// 注入端口号
    private int port;
    
    private String getBaseUrl() {
        return "http://localhost:" + port;
    }
    
    @Test 
    public void test() {
        Subject s = new Subject();
        s.setCreator(1l);
        s.setCreated(new Date());
        s.setSubjectName("test5");
        s.setDescr("test subject");
        s.setDeleted(0);
        s.setParent_id(0);
        s.setStatus(0);
        s.setType(0);
        String url = getBaseUrl() + "/subject/save";
        String result = template.postForObject(url, s, String.class);
    }
}

```

2. Transform objects with guava
http://www.leveluplunch.com/java/tutorials/002-transform-objects-with-guava/
All the function is a specific type of class that has an apply method that can be overridden. In the example, the apply method accepts a string as a parameter and returns an integer or the length of the string. Scrolling a bit further in the documentation, the most common use of functions is transforming collections.
```
Function<String, Integer> lengthFunction = new Function<String, Integer>() {
  public Integer apply(String string) {
    return string.length();
  }
};
```


Convert string to Enum

[2:16]

I have seeded some data so we can get right to the examples and you don't have to watch me type. The first example to show is transforming a string to Enum. Enums are quite popular and you may want to transform as it might be stored as a string in a database or some value.

Taking a look at the Day enum we created, it is a simple class that represents the days of the week:
```
public enum Day {

    SUNDAY, MONDAY, TUESDAY, WEDNESDAY,
    THURSDAY, FRIDAY, SATURDAY;   
}
```
We may have a list of strings with various strings representing days, Wednesday, Sunday, Monday... What we want to do is convert the list of strings to enums. There is a function in the Enum utility class in guava valueOfFunction that allows you to pass in the enum you want to convert to. In our case, Day.Class. Since we just upgraded to Guava 16.0, the valueOfFunction has been deprecated in preference of stringConverter. The stringConverter will return a function that converts a string to an Enum. Now we have a function that can be passed to guava utilities, in this case Iterables utility class by calling the transform method which will convert each string of list to a day enum.
```
@Test
public void transform_string_to_enum () {
    
    List<String> days = Lists.newArrayList(
            "WEDNESDAY", 
            "SUNDAY", 
            "MONDAY", 
            "WEDNESDAY");
    
    Function<String, Day> stringToDayEnum = Enums.stringConverter(Day.class);
    
    Iterable<Day> daysAsEnum = Iterables.transform(days, stringToDayEnum);
    
    for (Day day : daysAsEnum) {
        System.out.println(day);
    }
}
```
Output
```
WEDNESDAY
SUNDAY
MONDAY
WEDNESDAY
```


Convert from one object to another

[6:43]

The next example will look at is how to convert one object to another. Quite often we need to go to a legacy system to get data and populate a new set of domains for our new system. Or we may get data from a web service or whatever it may be. For this example, I created two objects. ETradeInvestment and TdAmeritradeInvestment which contain similar attributes of different types.
```
public class ETradeInvestment {
    
    private String key;
    private String name;
    private BigDecimal price;

    ...
}

public class TdAmeritradeInvestment {
    
    private int investmentKey;
    private String investmentName;
    private double investmentPrice;

    ...
}
```
There is a number of other utilities such as BeanUtils.copyProperties in apache commons or written your own if statements and have made it work that way. If we don't have a list of objects, we can call a method on function that will return a just a single object. We want to create the function that will map the TdAmeritradeInvestment to ETradeInvestment. If you ever get lost, you can use code assist or just remember the <F, T> just means from object, to object. For each iteration of the list, a new ETradeInvestment object will be created while mapping the TdAmeritradeInvestment to it. For the key, we will use the Ints.stringConverter which allows us to convert from a string to an int.

If we want to get some reuse out of this function, we could extract it outside this method or we can just use it inline. We will use the Lists utility to transform the list, above we use Iterables and there is also FluentIterables and Collections2. There is a lot of different ways which guava provides to use a function. If we run this code, we have EtradeInvestment object with the key, name and price. As a disclosure, I don't own any of these investments and were pulled from Google home page under the trending section. That did it, we converted from the TdAmeritradeInvestment to ETradeInvestment.
```
@Test
public void convert_tdinvestment_etradeinvestment () {
    
    List<TdAmeritradeInvestment> tdInvestments = Lists.newArrayList();
    tdInvestments.add(new TdAmeritradeInvestment(555, "Facebook Inc", 57.51));
    tdInvestments.add(new TdAmeritradeInvestment(123, "Micron Technology, Inc.", 21.29));
    tdInvestments.add(new TdAmeritradeInvestment(456, "Ford Motor Company", 15.31));
    tdInvestments.add(new TdAmeritradeInvestment(236, "Sirius XM Holdings Inc", 3.60));
    
    // convert a list of objects
    Function<TdAmeritradeInvestment, ETradeInvestment> tdToEtradeFunction = new Function<TdAmeritradeInvestment, ETradeInvestment>() {

        public ETradeInvestment apply(TdAmeritradeInvestment input) {
            ETradeInvestment investment = new ETradeInvestment();
            investment.setKey(Ints.stringConverter().reverse()
                    .convert(input.getInvestmentKey()));
            investment.setName(input.getInvestmentName());
            investment.setPrice(new BigDecimal(input.getInvestmentPrice()));
            return investment;
        }
    };

    List<ETradeInvestment> etradeInvestments = Lists.transform(tdInvestments, tdToEtradeFunction);
    
    System.out.println(etradeInvestments);
}
```
Output
```
[
ETradeInvestment{key=555, name=Facebook Inc, price=57.50},
ETradeInvestment{key=123, name=Micron Technology, Inc., price=21.28}, 
ETradeInvestment{key=456, name=Ford Motor Company, price=15.31}, 
ETradeInvestment{key=236, name=Sirius XM Holdings Inc, price=3.60}
]
```


Convert an object

[12:31]

If you have one single investment or object you want to convert, you can call the apply method directly on the function that will do the conversion.

ETradeInvestment faceBookInvestment = tdToEtradeFunction
                .apply(new TdAmeritradeInvestment(555, "Facebook Inc", 57.51));
Output
```
ETradeInvestment{key=555, name=Facebook Inc, price=57.50}
```


Convert list to map

[13:40]

One other really neat way to use function convert a list to a map. You may want to look up an object based on some object property. In this example, we want to create a map of TdAmeritradeInvestment based on the investment key. Taking a look we can use the Maps utility calling the uniqueIndex method which accepts a list and a function. The function, or the keyfunction, is used to produce the key for each value in the iterable. In this instance, the function will map from a TdAmeritradeInvestment and return an integer which will represent the key in the map.
```
@Test
public void transform_list_to_map () {
    
    List<TdAmeritradeInvestment> tdInvestments = Lists.newArrayList();
    tdInvestments.add(new TdAmeritradeInvestment(555, "Facebook Inc", 57.51));
    tdInvestments.add(new TdAmeritradeInvestment(123, "Micron Technology, Inc.", 21.29));
    tdInvestments.add(new TdAmeritradeInvestment(456, "Ford Motor Company", 15.31));
    tdInvestments.add(new TdAmeritradeInvestment(236, "Sirius XM Holdings Inc", 3.60));
    
    ImmutableMap<Integer, TdAmeritradeInvestment> investmentMap = Maps
            .uniqueIndex(tdInvestments,
                    new Function<TdAmeritradeInvestment, Integer>() {

                        public Integer apply(TdAmeritradeInvestment input) {
                            return new Integer(input.getInvestmentKey());
                        }
                    });
    
    System.out.println(investmentMap);
    
}
````
Output
```
{
555=TdAmeritradeInvestment{key=555, name=Facebook Inc, price=57.51}, 
123=TdAmeritradeInvestment{key=123, name=Micron Technology, Inc., price=21.29}, 
456=TdAmeritradeInvestment{key=456, name=Ford Motor Company, price=15.31}, 
236=TdAmeritradeInvestment{key=236, name=Sirius XM Holdings Inc, price=3.6}
}
```
Thanks for joining today's leveluplunch, have a great day.


3. How to convert a Java object (bean) to key-value pairs (and vice versa)?

Lots of potential solutions, but let's add just one more. Use Jackson (JSON processing lib) to do "json-less" conversion, like
```
ObjectMapper m = new ObjectMapper();
Map<String,Object> props = m.convertValue(myBean, Map.class);
MyBean anotherBean = m.convertValue(props, MyBean.class);
```

4. Integration testing on REST urls with Spring Boot

We are building a Spring Boot application with a REST interface and at some point we wanted to test our REST interface, and if possible, integrate this testing with our regular unit tests. One way of doing this, would be to @Autowire our REST controllers and call our endpoints using that. However, this won’t give full converage, since it will skip things like JSON deserialisation and global exception handling. So the ideal situation for us would be to start our application when the unit test start, and close it again, after the last unit test.
It just so happens that Spring Boot does this all for us with one annotation: @IntegrationTest.
Here is an example implementation of an abstract class you can use for your unit-tests which will automatically start the application prior to starting your unit tests, caching it, and close it again at the end.

```
package demo;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;

import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.test.IntegrationTest;
import org.springframework.boot.test.SpringApplicationConfiguration;
import org.springframework.boot.test.TestRestTemplate;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.HttpMethod;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
import org.springframework.test.context.web.WebAppConfiguration;

import com.fasterxml.jackson.databind.ObjectMapper;

@RunWith(SpringJUnit4ClassRunner.class)
// Your spring configuration class containing the @EnableAutoConfiguration
// annotation
@SpringApplicationConfiguration(classes = Application.class)
// Makes sure the application starts at a random free port, caches it throughout
// all unit tests, and closes it again at the end.
@IntegrationTest("server.port:0")
@WebAppConfiguration
public abstract class AbstractIntegrationTest {

    // Will contain the random free port number
    @Value("${local.server.port}")
    private int port;

    /**
     * Returns the base url for your rest interface
     * 
     * @return
     */
    private String getBaseUrl() {
        return "http://localhost:" + port;
    }

    // Some convenience methods to help you interact with your rest interface

    /**
     * @param requestMappingUrl
     *            should be exactly the same as defined in your RequestMapping
     *            value attribute (including the parameters in {})
     *            RequestMapping(value = yourRestUrl)
     * @param serviceReturnTypeClass
     *            should be the the return type of the service
     * @param parametersInOrderOfAppearance
     *            should be the parameters of the requestMappingUrl ({}) in
     *            order of appearance
     * @return the result of the service, or null on error
     */
    protected <T> T getEntity(final String requestMappingUrl, final Class<T> serviceReturnTypeClass, final Object... parametersInOrderOfAppearance) {
        // Make a rest template do do the service call
        final TestRestTemplate restTemplate = new TestRestTemplate();
        // Add correct headers, none for this example
        final HttpEntity<String> requestEntity = new HttpEntity<String>(new HttpHeaders());
        try {
            // Do a call the the url
            final ResponseEntity<T> entity = restTemplate.exchange(getBaseUrl() + requestMappingUrl, HttpMethod.GET, requestEntity, serviceReturnTypeClass,
                    parametersInOrderOfAppearance);
            // Return result
            return entity.getBody();
        } catch (final Exception ex) {
            // Handle exceptions
        }
        return null;
    }

    /**
     * @param requestMappingUrl
     *            should be exactly the same as defined in your RequestMapping
     *            value attribute (including the parameters in {})
     *            RequestMapping(value = yourRestUrl)
     * @param serviceListReturnTypeClass
     *            should be the the generic type of the list the service
     *            returns, eg: List<serviceListReturnTypeClass>
     * @param parametersInOrderOfAppearance
     *            should be the parameters of the requestMappingUrl ({}) in
     *            order of appearance
     * @return the result of the service, or null on error
     */
    protected <T> List<T> getList(final String requestMappingUrl, final Class<T> serviceListReturnTypeClass, final Object... parametersInOrderOfAppearance) {
        final ObjectMapper mapper = new ObjectMapper();
        final TestRestTemplate restTemplate = new TestRestTemplate();
        final HttpEntity<String> requestEntity = new HttpEntity<String>(new HttpHeaders());
        try {
            // Retrieve list
            final ResponseEntity<List> entity = restTemplate.exchange(getBaseUrl() + requestMappingUrl, HttpMethod.GET, requestEntity, List.class, parametersInOrderOfAppearance);
            final List<Map<String, String>> entries = entity.getBody();
            final List<T> returnList = new ArrayList<T>();
            for (final Map<String, String> entry : entries) {
                // Fill return list with converted objects
                returnList.add(mapper.convertValue(entry, serviceListReturnTypeClass));
            }
            return returnList;
        } catch (final Exception ex) {
            // Handle exceptions
        }
        return null;
    }

    /**
     * 
     * @param requestMappingUrl
     *            should be exactly the same as defined in your RequestMapping
     *            value attribute (including the parameters in {})
     *            RequestMapping(value = yourRestUrl)
     * @param serviceReturnTypeClass
     *            should be the the return type of the service
     * @param objectToPost
     *            Object that will be posted to the url
     * @return
     */
    protected <T> T postEntity(final String requestMappingUrl, final Class<T> serviceReturnTypeClass, final Object objectToPost) {
        final TestRestTemplate restTemplate = new TestRestTemplate();
        final ObjectMapper mapper = new ObjectMapper();
        try {
            final HttpEntity<String> requestEntity = new HttpEntity<String>(mapper.writeValueAsString(objectToPost));
            final ResponseEntity<T> entity = restTemplate.postForEntity(getBaseUrl() + requestMappingUrl, requestEntity, serviceReturnTypeClass);
            return entity.getBody();
        } catch (final Exception ex) {
            // Handle exceptions
        }
        return null;
    }
}
```
This entry was posted in Coding, REST, Spring, Testing and tagged rest, Spring Boot, spring integration by Ties van de Ven. Bookmark the permalink.

5. gg
6. gg
7. gg
8. gg
9. gg
10. gg
11. 