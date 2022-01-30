# Increase coverage in the API

If you run the test coverage like explained in the last section, you'll see
the test code coverage for the API.

```terminal
ok  	github.com/renato0307/learning-go-api	0.225s	coverage: 37.1% of statements
ok  	github.com/renato0307/learning-go-api/internal/apierror	0.378s	coverage: 100.0% of statements
?   	github.com/renato0307/learning-go-api/internal/apitesting	[no test files]
ok  	github.com/renato0307/learning-go-api/internal/middleware	0.863s	coverage: 94.0% of statements
ok  	github.com/renato0307/learning-go-api/pkg/finance	0.377s	coverage: 100.0% of statements
ok  	github.com/renato0307/learning-go-api/pkg/programming	0.565s	coverage: 89.3% of statements
```

The worst package is the `main`, with only 37.1% of test coverage.

We need to increase the coverage of the `main` function. To better do this, a
simple refactoring is needed: move the code to a dedicated function, named
`configureGin`:

```go
// ...

func main() {
	r := configureGin()
	r.Run()
}

func configureGin() *gin.Engine {
	// Initialize Gin
	gin.SetMode(gin.ReleaseMode)
	r := gin.New()
	r.Use(middleware.DefaultStructuredLogger())
	r.Use(middleware.Authenticator(newAuthenticatorConfig()))
	r.Use(middleware.Authorizer())
	r.Use(gin.Recovery())

	// Default route>
	r.GET("/", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "Hello, welcome to the learning-go-api",
		})
	})

	// Utility functions routes
	base := r.Group("/v1")

	p := programminglib.ProgrammingFunctions{}
	programming.SetRouterGroup(&p, base)

	useDefaultUrl := ""
	apiKey := getRequiredEnv(CURRCONV_API_KEY)
	f := financelib.NewFinanceFunctions(useDefaultUrl, apiKey)
	finance.SetRouterGroup(&f, base)

	return r
}
```

Next we implement the test for this function:

```go
func TestConfigureGin(t *testing.T) {
	// arrange
	setupFakeAuthServer()
	os.Setenv(CURRCONV_API_KEY, "fake_key")

	// act
	r := configureGin()

	// assert
	assert.NotNil(t, r)
	assert.Len(t, r.RouterGroup.Handlers, 4)
}

func TestGetRoot(t *testing.T) {
	// arrange
	setupFakeAuthServer()
	os.Setenv(CURRCONV_API_KEY, "fake_key")
	r := configureGin()

	w := httptest.NewRecorder()

	req, _ := http.NewRequest("GET", "/", nil)

	// act
	r.ServeHTTP(w, req)

	// assert
	assert.Equal(t, w.Code, http.StatusOK)
}
```

With this we reached more than 70% of code coverage.

To increase a bit further we can cover one the the error scenarios in the 
`newAuthenticatorConfig` config function: 

```go
func TestNewAuthenticatorWithInvalidTokenUrl(t *testing.T) {
	// arrange
	setupFakeAuthServer()
	os.Setenv(AUTH_JWKS_LOCATION, "invalid_url")

	// act & assert
	assert.Panics(t, func() {
		newAuthenticatorConfig()
	})
}

func setupFakeAuthServer() (string, string) {
	issuer := "https://cognito-idp.$AWS_REGION.amazonaws.com/$POOL_ID"
	sampleJwks := `
	{
		"keys": [{
			"kid": "1234example=",
			"alg": "RS256",
			"kty": "RSA",
			"e": "AQAB",
			"n": "1234567890",
			"use": "sig"
		}, {
			"kid": "5678example=",
			"alg": "RS256",
			"kty": "RSA",
			"e": "AQAB",
			"n": "987654321",
			"use": "sig"
		}]
	}`

	svr := httptest.NewServer(
		http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			fmt.Fprint(w, sampleJwks)
		}))

	os.Setenv(AUTH_TOKEN_ISS, issuer)
	os.Setenv(AUTH_JWKS_LOCATION, svr.URL)
	return issuer, sampleJwks
}
```

With this lates change we reach more than 85% of test coverage.

## Wrap up

Commit, push and create a new tag:

```sh
git add .
git commit -m "test: increase coverage"
git push
git tag -a v0.0.6 -m "v0.0.6"
git push origin v0.0.6
```

# Next
 
The next section is
[Increase coverage in the CLI](it8-cli-test-code-coverage.md).