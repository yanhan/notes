# About

Some general principes on Golang programming, esp when writing servers.


## Style

Follow this / adapt it to your needs: https://github.com/uber-go/guide/blob/master/style.md


## General rules

1. Expose interfaces instead of concrete structs. This will make it possible to mock
  - If using a function from an external library / similar that has side effects which need to be mocked in a test, do not call the function directly. Instead, create an interface with a method for it. Then write a struct which implementats that interface. Inject that interface into where it is required. This allows for mocking
2. Dependency injection
3. If a function takes in more than 3 parameters, create a struct to house the parameters. Similarly for return values
4. Error logs: log where the error occurs and in the case of a server, before the response is returned. Do not log when the error is propagated upwards
5. Do not chase test coverage. Instead, aim to at least have sufficient tests so that you will be confident to do major refactors
6. It is much more useful to have tests that use a real database compared to mocking out the DB layer all the time. This allows you to check that data is stored correctly
7. For functions which are used very often, write tests minimally for its major code paths
8. Configuration: try not to use it to store data structures that will be shared across multiple environments. It is probably better to store them in code, or in another file which can be loaded
9. Pointer fields in structs: if the use case is to indicate if a field is present, do it sparingly. Otherwise your code will be littered with many nil checks / similar and increases noise
10. If there is a 'stronger' typed option (eg. defining a concrete struct vs map), go for that unless it is too cumbersome / not practical
11. Goroutines: if a function should only be called within a goroutine, do add a comment to indicate it


## Useful libraries for testing

- https://github.com/stretchr/testify
- https://github.com/golang/mock
- https://github.com/vektra/mockery
