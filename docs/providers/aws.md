# AWS

This section is all about using, debugging and extending the official CloudQuery AWS Provider 





Authentication with AWS:

Using the go sdk we support the entire AWS Credential Chain including:
- Environment Variables AWS_*
- Shared Config file stored at `~/.aws/` or where ever the `aws_shared_credentials_file` variable is pointing
- Container/Instance metadata providers



Building the Provider:

``` bash
make build
```


Running locally:

1. Start a local database:
    ```
    make pg-start
    ```
2. Configure the `config.hcl`
    ```
    cloudquery init aws
    ```
3. open the `config.hcl`

```
make run
```