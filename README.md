# Spring Cloud Contract (Using Maven)

References taken from [baeldung](https://www.baeldung.com/spring-cloud-contract)

## Setup

### Controller

```java
@RestController
public class EvenOddController {
 
    @GetMapping("/validate/prime-number")
    public String isNumberPrime(@RequestParam("number") Integer number) {
        return Integer.parseInt(number) % 2 == 0 ? "Even" : "Odd";
    }
}
```

### Maven Plugin & Dependency

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <version>2.1.1.RELEASE</version>
    <scope>test</scope>
</dependency>
```

```xml
<plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>2.1.1.RELEASE</version>
    <extensions>true</extensions>
    <configuration>
        <baseClassForTests>
            com.example.cloudcontract.BaseTestClass
        </baseClassForTests>
    </configuration>
</plugin>
```

### BaseTestCase

```java
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.MOCK)
@DirtiesContext
@AutoConfigureMessageVerifier
public class BaseTestClass {
 
    @Autowired
    private EvenOddController evenOddController;
 
    @Before
    public void setup() {
        StandaloneMockMvcBuilder standaloneMockMvcBuilder 
          = MockMvcBuilders.standaloneSetup(evenOddController);
        RestAssuredMockMvc.standaloneSetup(standaloneMockMvcBuilder);
    }
}
```

In the /src/test/resources/contracts/ package, we'll add the test stubs, such as this one in the file shouldReturnEvenWhenRequestParamIsEven.groovy:

```groovy
import org.springframework.cloud.contract.spec.Contract
Contract.make {
    description "should return even when number input is even"
    request{
        method GET()
        url("/validate/prime-number") {
            queryParameters {
                parameter("number", "2")
            }
        }
    }
    response {
        body("Even")
        status 200
    }
}
```

When we run the build, the plugin automatically generates a test class named ContractVerifierTest that extends our BaseTestClass and puts it in /target/generated-test-sources/contracts/.

The names of the test methods are derived from the prefix “validate_” concatenated with the names of our Groovy test stubs. For the above Groovy file, the generated method name will be “validate_shouldReturnEvenWhenRequestParamIsEven”.

Let's have a look at this auto-generated test class:

```java
public class ContractVerifierTest extends BaseTestClass {
 
    @Test
    public void validate_shouldReturnEvenWhenRequestParamIsEven() throws Exception {
        // given:
        MockMvcRequestSpecification request = given();
     
        // when:
        ResponseOptions response = given().spec(request)
          .queryParam("number","2")
          .get("/validate/prime-number");
     
        // then:
        assertThat(response.statusCode()).isEqualTo(200);
         
        // and:
        String responseBody = response.getBody().asString();
        assertThat(responseBody).isEqualTo("Even");
    }
}
```

The build will also add the stub jar in our local Maven repository so that it can be used by our consumer.

Stubs will be present in the output folder under stubs/mapping/.

Sample Stub

```json
{
  "id" : "38b254eb-d541-4bc6-a5a4-8d7cae204dc0",
  "request" : {
    "urlPath" : "/validate/prime-number",
    "method" : "GET",
    "queryParameters" : {
      "number" : {
        "equalTo" : "2"
      }
    }
  },
  "response" : {
    "status" : 200,
    "body" : "Even",
    "transformers" : [ "response-template" ]
  },
  "uuid" : "38b254eb-d541-4bc6-a5a4-8d7cae204dc0"
}
```