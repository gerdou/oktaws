# oktaws

A CLI tool to obtain AWS credentials using Okta. Supports both OIDC device authorization and a browser-based SAML flow.

## Features

- OIDC device authorization flow (Okta device code)
- Browser-based SAML flow with a local callback server and a Chrome/Firefox extension
- Configuration via YAML file, environment variables, or CLI flags
- Writes AWS credentials to `~/.aws/credentials` using the selected profile (default: `default`)
- Optional access-token cache file for OIDC (`~/.okta/awscli/cache.json`)
- Debug and API debug logging

## Installation

### Homebrew

```bash
brew tap gerdou/oktaws
brew install oktaws
```

### Build from source

```bash
go build -o oktaws
```

## Quick Start

### 1. Initialize Configuration

```bash
./oktaws config init
```

This creates `~/.config/oktaws/config.yaml` with defaults and prompts.

### 2. Set Your Okta Configuration

For SAML browser flow:

```bash
./oktaws config set org_domain your-org.okta.com
./oktaws config set aws_acct_fed_app_id exkXXXXXXXXXXXXXXXX
```

For OIDC device flow:

```bash
./oktaws config set org_domain your-org.okta.com
./oktaws config set oidc_client_id your-client-id
```

### 3. Authenticate

```bash
./oktaws
```

With `auth_flow: auto`, the CLI chooses:

1. OIDC if `oidc_client_id` is set
2. SAML browser flow if `aws_acct_fed_app_id` is set
3. OIDC if neither is set

On success, credentials are written to `~/.aws/credentials` under the configured profile.

## Configuration

### Configuration File

Location: `~/.config/oktaws/config.yaml`

```yaml
auth_flow: auto
org_domain: your-org.okta.com
oidc_client_id: ""
aws_acct_fed_app_id: exkXXXXXXXXXXXXXXXX
aws_iam_role: ""
aws_iam_idp: ""
aws_region: us-east-1
session_duration: 3600
profile: default
format: json
open_browser: false
open_browser_command: ""
write_aws_credentials: false
cache_access_token: false
qr_code: false
all_profiles: false
debug: false
debug_api_calls: false
```

Valid `format` values are `json` and `env`. See the Output section for current behavior.

### Configuration Commands

```bash
# View current configuration
./oktaws config list

# Get a specific value
./oktaws config get org_domain

# Set a value
./oktaws config set org_domain your-org.okta.com

# Show config file path
./oktaws config path
```

### Configuration Priority

1. CLI flags (highest priority)
2. Environment variables
3. Config file
4. Defaults (lowest priority)

### Environment Variables

The following environment variables are read as defaults for string flags:

- `OKTA_AWSCLI_ORG_DOMAIN`
- `OKTA_AWSCLI_OIDC_CLIENT_ID`
- `OKTA_AWSCLI_IAM_ROLE`
- `OKTA_AWSCLI_IAM_IDP`
- `OKTA_AWSCLI_AWS_ACCOUNT_FEDERATION_APP_ID`
- `OKTA_AWSCLI_PROFILE`
- `OKTA_AWSCLI_SESSION_DURATION`
- `OKTA_AWSCLI_FORMAT`
- `OKTA_AWSCLI_AWS_REGION`
- `OKTA_AWSCLI_BROWSER_COMMAND`

## Authentication Flows

### SAML Browser Flow (Recommended for interactive use)

```bash
./oktaws --auth-flow saml-browser
```

Best for:
- Users with a standard Okta AWS SAML application
- Interactive, browser-based workflows

How it works:
1. Starts a local callback server on `127.0.0.1:8765`
2. Ensures the browser extension is installed
3. Opens your browser to Okta
4. Extension captures the SAML assertion and POSTs it to the CLI
5. CLI exchanges SAML for AWS STS credentials
6. Credentials are written to `~/.aws/credentials`

### OIDC Device Flow

```bash
./oktaws --auth-flow oidc --oidc-client-id your-client-id
```

Best for:
- CI/CD or headless environments
- When you have an OIDC client configured in Okta

How it works:
1. Requests a device code from Okta
2. Prints the verification URL and user code
3. Optionally opens the browser when `--open-browser` is set
4. Polls for an access token
5. Uses the token to fetch a SAML assertion
6. Exchanges SAML for AWS STS credentials
7. Credentials are written to `~/.aws/credentials`

Optional:
- `--cache-access-token` writes a cache file to `~/.okta/awscli/cache.json`

## CLI Flags

### Authentication

- `--auth-flow`, `-x`: `auto`, `oidc`, or `saml-browser` (default: `auto`)
- `--org-domain`, `-o`: Okta organization domain
- `--oidc-client-id`, `-c`: OIDC client ID (required for OIDC flow)
- `--aws-acct-fed-app-id`, `-a`: AWS Account Federation app ID (required for SAML flow)
- `--aws-iam-role`, `-r`: AWS IAM role ARN or substring (used to pick a role)
- `--aws-iam-idp`, `-i`: AWS IAM identity provider ARN (currently unused)

### AWS Configuration

- `--aws-region`, `-n`: AWS region (config init defaults to `us-east-1`)
- `--aws-session-duration`, `-s`: Session duration in seconds (default: `3600`)

### Output

- `--profile`, `-p`: AWS profile name (default: `default`)
- `--format`, `-f`: Output format `json` or `env`
- `--write-aws-credentials`, `-w`: Write to `~/.aws/credentials` (see Output section)
- `--all-profiles`, `-k`: Collect all profiles (currently unused)

### Browser

- `--open-browser`, `-b`: Open browser automatically for OIDC device flow (default: `false`)
- `--open-browser-command`, `-m`: Custom browser command

### Other

- `--qr-code`, `-q`: Display QR code (currently unused)
- `--cache-access-token`, `-e`: Cache access token to `~/.okta/awscli/cache.json`
- `--debug`, `-g`: Enable debug output
- `--debug-api-calls`, `-d`: Debug Okta API calls
- `--version`, `-v`: Print version and exit

## Output

By default, oktaws writes credentials to `~/.aws/credentials` under the selected profile and exits. The profile defaults to `default`.

The `--format` flag only applies when credentials are not written to file. With the current defaults, the file write path is always taken because a profile is always set. If you need stdout output (`json` or `env`), you will need to clear the profile in configuration and disable `--write-aws-credentials`.

### AWS Credentials File

```bash
./oktaws --write-aws-credentials --profile my-profile
```

Writes to `~/.aws/credentials`:

```ini
[my-profile]
aws_access_key_id = ASIA...
aws_secret_access_key = ...
aws_session_token = ...
```

### JSON (stdout)

```bash
./oktaws --format json
```

### Environment Variables (stdout)

```bash
./oktaws --format env
```

Use with:

```bash
eval $(./oktaws --format env)
```

Note: Stdout output is only used when file writing is disabled as described above.

## Browser Extension

The SAML browser flow requires a browser extension that captures SAML assertions.

### First-Time Setup

When you first run with `--auth-flow saml-browser`, the CLI will:

1. Detect your browser (Chrome or Firefox)
2. Extract extension files to `~/.config/oktaws/extension/`
3. Attempt to enable Chrome Developer Mode (Chrome only)
4. Open the extensions page in your browser
5. Guide you through installation

Firefox note: the installer uses `about:debugging` and loads a temporary add-on, which must be reloaded after Firefox restarts.

### Supported Browsers

- Google Chrome
- Mozilla Firefox

### Extension Permissions

The extension requires:

- Okta domains: `*.okta.com`, `*.okta-emea.com`, `*.oktapreview.com`
- AWS sign-in and console: `*.signin.aws.amazon.com`, `*.console.aws.amazon.com`
- Localhost access: `http://localhost:*`
- Network request interception and storage

## Finding Your Configuration Values

### Okta Organization Domain

Your Okta domain is in the URL when you log into Okta:

```
https://your-org.okta.com
         ^^^^^^^^^^^^^^^^
```

### AWS Account Federation App ID

1. Log into Okta
2. Navigate to your AWS tile
3. Look at the URL:

```
https://your-org.okta.com/app/amazon_aws/exkXXXXXXXXXXXXXXXX/sso/saml
                                         ^^^^^^^^^^^^^^^^^^^^
```

### OIDC Client ID (if using OIDC flow)

Ask your Okta administrator to create an OIDC client for you and provide the client ID.

## Examples

### Example 1: Quick credentials for default profile

```bash
# Configure once
./oktaws config set org_domain my-company.okta.com
./oktaws config set aws_acct_fed_app_id exk123456789

# Get credentials (writes to ~/.aws/credentials)
./oktaws

# Use AWS CLI
aws s3 ls --profile default
```

### Example 2: OIDC device flow

```bash
./oktaws --auth-flow oidc --oidc-client-id 0oa123456789 --open-browser --cache-access-token
aws sts get-caller-identity --profile default
```

### Example 3: Multiple AWS profiles

```bash
# Dev account
./oktaws --profile dev --aws-acct-fed-app-id exkDEV123 --write-aws-credentials

# Prod account
./oktaws --profile prod --aws-acct-fed-app-id exkPROD456 --write-aws-credentials

# Use profiles
aws s3 ls --profile dev
aws s3 ls --profile prod
```

### Example 4: Long-lived session

```bash
./oktaws --aws-session-duration 43200  # 12 hours
```

### Example 5: Specific role selection

```bash
./oktaws --aws-iam-role admin-role
```

## Troubleshooting

### Extension not detected

Issue: CLI says extension is not installed, but you installed it.

Solution:
- Make sure Chrome or Firefox is running
- Verify the extension is enabled
- The extension name should be "Oktaws SAML Interceptor"
- Try restarting the browser after installation

### Port already in use

Issue: `bind: address already in use` on port 8765

Solution:

```bash
# Kill the process using the port
lsof -ti:8765 | xargs kill -9
```

### SAML not captured

Issue: Browser opens but SAML is never received.

Solution:
- Check browser console for `[Oktaws]` messages
- Verify extension is loaded
- Try reloading the extension
- Make sure you're authenticating (not already logged into AWS)

### Extension permissions error

Issue: Extension needs additional permissions.

Solution:
- Click the extension icon in your browser
- Grant the requested permissions
- Refresh the Okta page

### Multiple roles available

Issue: You have access to multiple AWS roles.

Solution:
- CLI will prompt you to select a role
- Or specify with `--aws-iam-role role-name`
- Add to config file to avoid prompts:

```yaml
aws_iam_role: admin-role
```

### Browser does not open

Issue: The browser does not open automatically during OIDC device flow.

Solution:
- Use `--open-browser` to enable auto-open
- Or set a custom browser command: `--open-browser-command "/path/to/browser"`

## Security Notes

- Credentials are written to `~/.aws/credentials` by default
- The extension only runs on Okta and AWS domains listed above
- The callback server only listens on localhost (`127.0.0.1`)
- SAML assertions are only used in memory
- Access token caching is optional and writes to `~/.okta/awscli/cache.json`

## Contributing

Issues and pull requests are welcome.

## License

MIT
