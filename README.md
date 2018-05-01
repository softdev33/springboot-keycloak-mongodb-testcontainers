# springboot-testing-mongodb-keycloak

## Goal

The goals of this project are:

- Create a REST API application to manage books, `book-service`. The data is stored in a `MongoDB` database.
The application will have its endpoints related to add/update/delete books secured.
- Use `Keycloak` as authentication and authorization server;
- Explore the utilities and annotations that Spring Boot provides when testing applications.

## Start Environment

***Note. In order to run some commands/scripts, you must have [`jq`][1] installed on you machine***

### Docker Compose

##### 1. Open one terminal

##### 2. Inside `/springboot-testing-mongodb-keycloak/dev` folder run
```
docker-compose up
```

### Configure Keycloak

You have two ways to configure `Keycloak`: running `init_keycloak.sh` script or manually using `Keycloak UI`.

#### Running `init_keycloak.sh` script

##### 1. Go `/springboot-testing-mongodb-keycloak/dev` folder

##### 2. Then run the following script to initialize `Keycloak`.
It will create automatically the `company-services` realm, `book-service` client, `manage_books` client role and the user `ivan.franchin` with the role `manage_books` assigned.  
```
./init_keycloak.sh
```

##### 3. `BOOKSERVICE_CLIENT_SECRET` is shown on the script outputs. Export it to an environment variable
```
export BOOKSERVICE_CLIENT_SECRET=...
```

#### Manually using `Keycloak UI`

##### 1. Access the link
```
http://localhost:8181
```

##### 2. Login with the credentials
```
Username: admin
Password: admin
```

##### 3. Create a new Realm
- Go to top-left corner and hover the mouse over `Master` realm. A blue button `Add realm` will appear. Click on it.
- On `Name` field, write `company-services`. Click on `Create`.

##### 4. Create a new Client
- Click on `Clients` menu on the left.
- Click `Create` button.
- On `Client ID` field type `book-service`.
- Click on `Save`.
- On `Settings` tab, set the `Access Type` to `confidential`.
- Still on `Settings` tab, set the `Valid Redirect URIs` to `http://localhost:8080/*`.
- Click on `Save`.
- Go to `Credentials` tab. Copy the value on `Secret` field. It will be used on the next steps.
- Go to `Roles` tab.
- Click `Add Role` button.
- On `Role Name` type `manage_books`.
- Click on `Save`.

##### 5. Create a new User
- Click on `Users` menu on the left.
- Click on `Add User` button.
- On `Username` field set `ivan.franchin`.
- Click on `Save`.
- Go to `Credentials` tab.
- Set to `New Password` and `Password Confirmation` the value `123`.
- Turn off the `Temporary` field.
- Click on `Reset password`.
- Confirm the pop up clicking on `Change Password`.
- Go to `Role Mappings` tab.
- Select `book-service` on the combo-box `Client Roles`.
- Add the role `manage_books` to `ivan.franchin`.

**Done!** That is all the configuration needed on `Keycloak`. 

### Start `book-service`

##### 1. Open a new terminal

##### 2. Start `springboot-testing-mongodb-keycloak` application

In `springboot-testing-mongodb-keycloak` root folder, run:

```
gradle bootRun 
```

## Test using cURL

##### 1. Open a new terminal

##### 2. Call the endpoint `GET /api/books` using the cURL command bellow.
```
curl -i 'http://localhost:8080/api/books'
```

It will return:
```
Code: 200
Response Body: []
```

##### 3. Try to call the endpoint `POST /api/books` using the cURL command bellow.
``` 
curl -i -X POST 'http://localhost:8080/api/books' \
  -H "Content-Type: application/json" \
  -d '{ "authorName": "ivan", "title": "java 8", "price": 10.5 }'
```
It will return:
```
Code: 302
```

Here, the application is trying to redirect the request to an authentication link.

##### 4. Run the commands bellow to get an access token for `ivan.franchin` user.

*PS. Update the `BOOKSERVICE_CLIENT_SECRET` value with the secret generated by `Keycloak` (Configure Keycloak)*

```
MY_ACCESS_TOKEN=$(curl -s -X POST \
  "http://localhost:8181/auth/realms/company-services/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=ivan.franchin" \
  -d "password=123" \
  -d "grant_type=password" \
  -d "client_secret=$BOOKSERVICE_CLIENT_SECRET" \
  -d "client_id=book-service" | jq -r .access_token)
```

##### 5. Call the endpoint `POST /api/books` using the cURL command bellow.
```
curl -i -X POST 'http://localhost:8080/api/books' \
  -H "Authorization: Bearer $MY_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "authorName": "ivan", "title": "java 8", "price": 10.5 }'
```

It will return:

```
Code: 201
Response Body: {
  "id":"01d984be-26bc-49f5-a201-602293d62b82",
  "authorName":"ivan",
  "title":"java 8",
  "price":10.5
}
```

## Running unit and integration testing

##### 1. In order to run unit and integration testing type
```
gradle test integrationTest
```

##### 2. From `springboot-testing-mongodb-keycloak` root folder, unit testing report can be found in
```
/build/reports/tests/test/index.html
```

##### 3. From `springboot-testing-mongodb-keycloak` root folder, integration testing report can be found in
```
/build/reports/tests/integrationTest/index.html
```

## More about testing Spring Boot Applications

Spring Boot provides a number of utilities and annotations to help when testing your application.

### Unit Testing

The idea of the unit testing is to test each layer of the application (repository, service and controller) individually.
The repository classes usually don't depends on any other classes, so we can write test cases without any mocking.
On the other hand, the services classes depend on repositories. So, as we have already test cases to cover the repositories, while writing test cases for the services we don't need to care about the quality of the repositories classes. So, every calls to repositories classes should be mocked.
The same happens to controller classes that depends on the services classes. While writing tests for the controllers, service calls on the controller classes should be mocked.

#### Repository Testing

You can use `@DataMongoTest` to test `MongoDB` applications.
By default, it configures an in-memory embedded `MongoDB` (if available), configures a `MongoTemplate`, scans for `@Document` classes, and configures Spring Data `MongoDB` repositories.
The embedded `MongoDB` added in the `build.gradle` and can by found in the link: https://github.com/flapdoodle-oss/de.flapdoodle.embed.mongo
Regular `@Component` beans are not loaded into the `ApplicationContext`.
An example of utilization is:

```
@RunWith(SpringRunner.class)
@DataMongoTest
public class ExampleRepositoryTests {

    @Autowired
    private MongoTemplate mongoTemplate;

	@Autowired
	private BookRepository bookRepository;

	// Tests ..
}
```

#### Service Testing

In order to test the application services, we can use a something similar as shown bellow, as we create an instance of `BookServiceImpl` and mock the `bookRepository` 

```
@RunWith(SpringRunner.class)
public class BookServiceImplTest {

    private BookService bookService;
    private BookRepository bookRepository;

    @Before
    public void setUp() {
        bookRepository = mock(BookRepository.class);
        bookService = new BookServiceImpl(bookRepository);
    }
    
    // Tests
}
```

#### Controller Testing

`@WebMvcTest` annotation can be used to test whether Spring MVC controllers are working as expected.
`@WebMvcTest` is limited to a single controller and is used in combination with `@MockBean` to provide mock implementations for required dependencies.
`@WebMvcTest` also auto-configures `MockMvc`. Mock MVC offers a powerful way to quickly test MVC controllers without needing to start a full HTTP server.
In the example bellow, you can see that we mocking the services (in this case `bookService`) used by `BookController`.

```
@RunWith(SpringRunner.class)
@WebMvcTest(BookController.class)
public class BookControllerTests {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private BookService bookService;
    
    // Tests ... 
}
```

#### DTO Testing

`@JsonTest` annotation can be used to test whether object JSON serialization and deserialization is working as expected.
In the example bellow, it is used `JacksonTester`. However, `GsonTester`, `JsonbTester` and `BasicJsonTester` could also be used instead.
Btw, I've tried to use all of them, but just `JacksonTester` worked easily and as expected.  

```
@RunWith(SpringRunner.class)
@JsonTest
public class MyJsonTests {

	@Autowired
	private JacksonTester<VehicleDetails> json;

	// Tests ...
}
```

### Integration Testing

The main goal of the integration tests is, as its name suggests, to integrate the different layers of the application. Here, no mocking is involved and a full running HTTP server is needed. 
So, in order to have it, we can use the `@SpringBootTest(webEnvironment=WebEnvironment.RANDOM_PORT)` annotation. What this annotation does is to start a full running server running in a random ports. Spring Boot also provides a `TestRestTemplate` facility, for example:

```
@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
public class RandomPortTestRestTemplateExampleTests {

	@Autowired
	private TestRestTemplate restTemplate;

	// Tests ...

}
```

Integration tests should run separated from the unit tests and, mainly, it should runs after unit tests. In this project, we created a new integrationTest Gradle task to handle exclusively integration tests.

### Sources:

- https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-testing.html
- http://www.baeldung.com/spring-boot-testing

[1]: https://stedolan.github.io/jq