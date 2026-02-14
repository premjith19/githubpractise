# Backend

## Development

It uses an in-memory H2 database.

Launch the application:

```shell
./mvnw spring-boot:run
```

## Production

### Prerequisite

Launch a PostgreSQL database.

E.g. with docker:

```shell
docker container run --name mca-postgres --env POSTGRES_PASSWORD=postgres --publish 5432:5432 --detach postgres
```

Set env vars if different from [defaults](./src/main/resources/application-prod.properties#1-3):

- `SPRING_POSTGRES_URL` in the form `jdbc:postgresql://${HOST}:${PORT}/${DB}`
- `SPRING_POSTGRES_USERNAME`
- `SPRING_POSTGRES_PASSWORD`

### Run

Build the application:

```shell
./mvnw package
```

Launch the application:

```shell
SPRING_PROFILES_ACTIVE=prod java -jar ./target/application-0.0.1-SNAPSHOT.jar
```

## Check

Visualize the API: http://localhost:8080/api/swagger-ui/index.html
