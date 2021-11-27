# Overview

In this tutorial, you will build a new CloudQuery Provider that will interact with GitHub API.

## Bootstrap New Provider

The best way to bootstrap a new provider is to use the [cq-provider-template](https://github.com/cloudquery/cq-provider-template).

You can either click `Use this template` in GitHub UI or clone the repo and reinitialize the git (as you don't need the template history).

```bashs
git clone https://github.com/cloudquery/cq-provider-template cq-provider-github
cd cq-provider-github
rm -rf .git
git init -b main
```

### Update cq-provider-template reference to your new provider name

There are a few places where you will need to update the template stubs (you can also `grep` or search for `CHANGEME` comment in the code):

`go.mod`, `main.go`
```
module github.com/cloudquery/cq-provider-template
# Change to 
module github.com/your_org_or_user/cq-provider-github
# we will use github.com/cloudquery/cq-provider-github in this tutorial
```

### Update Provider Name

The provider name is the name you will use when you will call `cloudquery init provider_name`.

Change `reosurces.go`:
```go
func Provider() *provider.Provider {
	return &provider.Provider{
		Name:      "github",
```

### Choose Go API Client

Usually, each provider will use one Go Client to interact with the service. As we need to load the data to relational database we will go with [google/go-github](https://github.com/google/go-github) that implements all GitHub RestAPIs.

Install `go-github`
```
go get github.com/google/go-github/v40
```

#### Define config.hcl block

To initialize an [authenticated](https://github.com/google/go-github#authentication) GitHub API client you will need an AccessToken provided by the user.

`client/config.go`
```go
type Config struct {
    // Add this LINE:
	GitHubToken string `hcl:"github_token,optional"`

	// resources that user asked to fetch
	// each resource can have optional additional configurations
	Resources []struct {
		Name  string
		Other map[string]interface{} `hcl:",inline"`
	}
}
func (c Config) Example() string {
	return `configuration {
    // Add this line    
	// api_key = ${your_env_variable}
    // api_key = static_api_key
}
`
```

Config will be automatically parsed by CloudQuery SDK, you just need to define the name via the `hcl` tag. We will also make it optional as we want also to support default environment variables.

The `Example` function returns an example `config.hcl` when you will run `cloudquery init github`. The `configuration` is mostly commented out as the api_key is optional (we want to support `GITHUB_TOKEN` env variable by default).

#### Initialize GitHub API Client

Following the example in [go-github](https://github.com/google/go-github#authentication) we need to initialize the client in `client/clieng.go`

```go
type Client struct {
	// This is a client that you need to create and initialize in Configure
	// It will be passed for each resource fetcher.
	logger hclog.Logger

    // Add this line
	GithubClient *github.Client
}

func (c *Client) Logger() hclog.Logger {
	return c.logger
}

func Configure(logger hclog.Logger, config interface{}) (schema.ClientMeta, error) {
	providerConfig := config.(*Config)
    // Start Snippet
	ctx := context.Background()
	token, exists := os.LookupEnv("GITHUB_TOKEN")
	if !exists {
		token = providerConfig.GitHubToken
	}
	ts := oauth2.StaticTokenSource(
		&oauth2.Token{AccessToken: token},
	)
	tc := oauth2.NewClient(ctx, ts)
	githubClient := github.NewClient(tc)
    // End Snippet

	// Init your client and 3rd party clients using the user's configuration
	// passed by the SDK providerConfig
	client := Client{
		logger: logger,
		// Add this line: pass the initialized third pard client
		GithubClient: githubClient,
	}

	// Return the initialized client and it will be passed to your resources
	return &client, nil
}
```

Configure is called once before starting an operation such as `fetch`. This is usually the place where you need to parse the user `configuration` and initialize the API Client.

In this case we first check if the token is available in `GITHUB_TOKEN` and if not we read what is available in the the parsed configuration. 

#### Add first resource

Now we are set to implement our first resource which will extract,transform and load configuration from GitHub to PostgreSQL.