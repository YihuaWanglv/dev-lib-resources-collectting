


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


2. gg
3. gg
4. gg
5. gg
6. gg
7. gg
8. gg
9. gg
10. gg
11. 