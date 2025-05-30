# QSS - Quick SSM Session

QSS (Quick SSM Session) is a command-line tool designed to streamline the process of connecting to EC2 instances using AWS Systems Manager (SSM) Session Manager. It acts as a user-friendly frontend to the AWS CLI, abstracting away much of the underlying complexity. If you find yourself frequently navigating multiple AWS accounts, regions, and environments, QSS can significantly simplify your workflow.

The core idea behind QSS is to let you connect to an instance using a memorable and logical hierarchy: an *account name*, an *environment name* (e.g., `staging`, `production`), and an *instance short name* (e.g., `app-server-01`). You no longer need to recall specific instance IDs, AWS regions, or profile names for each connection.

One of its standout features is its powerful **bash completion** support. This not only speeds up command entry for accounts, environments, and instance short names but also allows you to interactively explore your available infrastructure. As you type, the QSS completion suggests valid options, making it easy to discover and select the instance you need.

QSS achieves this by maintaining a local database of your EC2 instances. This database indexes instances by their account, environment, and short name. Environment and short names are dynamically extracted from your actual EC2 instance names (from the "Name" tag) using customizable regular expressions. This means consistent instance naming conventions are key to leveraging QSS effectively. The local database can be easily refreshed, either entirely or per account/environment, ensuring it stays up-to-date with your cloud resources.

A key design choice for robustness is that the local database primarily stores instance *names* rather than fixed instance IDs. The actual instance ID is resolved dynamically at connection time (or when requesting info). This approach gracefully handles scenarios where instances are re-created by Auto Scaling Groups, as their names often persist while their IDs change.

Beyond just connecting, QSS allows you to:
- List instances from its local database.
- Display detailed information about indexed instances by querying EC2 in real-time (state, IDs, IPs, tags, and more).

Highly customizable through a JSON configuration file (`~/.qss.json` by default), QSS lets you define your accounts, AWS profiles, regions, naming templates, and even the exact connection command. If you're looking for a way to make SSM sessions faster, more intuitive, and less error-prone across complex AWS setups, QSS might be the tool for you.

## Features

-   **Quick Connections**: Easily connect to EC2 instances via AWS SSM Session Manager using `account/environment/short-name` identifiers.
-   **Powerful Bash Completion**: Interactive command-line completion for accounts, environments, and instance names.
-   **Local Instance Database**: Maintains a local, quickly refreshable JSON database of your instances, indexed for fast lookups.
-   **Dynamic Name & Environment Parsing**: Automatically extracts environment and short names from EC2 instance `Name` tags using user-defined regex templates.
-   **Robust Instance ID Handling**: Resolves instance IDs at connection time, making it resilient to instance re-creation by Auto Scaling Groups.
-   **Instance Listing**: List instances from the local database with flexible grouping by region or environment.
-   **Detailed Instance Information**: Display comprehensive, real-time EC2 instance details (state, IDs, IPs, tags, etc.) directly from AWS.
-   **Multi-Account & Multi-Region**: Designed from the ground up for users working across numerous AWS accounts and regions.
-   **Highly Customizable**: Extensive configuration options via a `~/.qss.json` file, including AWS profiles, regions, naming templates, and custom connection commands.
-   **AWS CLI Frontend**: Acts as an intelligent and user-friendly layer on top of the AWS CLI for SSM session management.
-   **Bash Completion Installation**: Includes a command to help install the bash completion script.

## Prerequisites

- Ruby 2.4 or higher
  _A Ruby version supporting recent releases of the `aws-sdk-ec2` gem is recommended for better compatibility and security._
- Ruby gems:
  - `aws-sdk-ec2` (for EC2 instance interactions)
  - `json-schema` (for configuration validation)
- AWS CLI installed and configured
- Session Manager plugin for AWS CLI
- AWS API access with permissions to list EC2 instances and establish SSM sessions

## Installation

1.  Copy the `qss` script, preferably in a directory present in your `$PATH`.
    To obtain the script, you may choose to clone the entire git repository:
    ```bash
    git clone https://github.com/bogbert/qss.git
    ```
    Then navigate into the `qss` directory or copy the `qss` script to your desired location.

2.  Make the script executable:
    ```bash
    chmod +x /path/to/your/qss
    ```

3.  Install required gems:
    ```bash
    gem install aws-sdk-ec2 json-schema
    ```

4.  Install the bash completion script (_optional but highly recommended_):
    ```bash
    /path/to/your/qss --install-completion
    ```
    The script will guide you through the installation process. It will prompt for the installation destination and may suggest an existing bash completion directory if one is detected. If the completion script isn't installed in a directory automatically sourced by your shell, you'll need to add a line to source it in your bash initialization file (e.g., `~/.bashrc` or `~/.bash_profile`), for example:
    ```bash
    source /path/to/your/qss_completion_script
    ```
    Remember to restart your shell or source your bash initialization file for the changes to take effect.

## Configuration

Create a configuration file at `~/.qss.json`. Here's an example:

```json
{
  "default_env": "noenv",
  "default_regions": ["eu-west-3"],
  "default_templates": ["std_env"],
  "accounts": {
    "dev": {
      "aws_profile": "dev-profile"
    },
    "prod": {
      "aws_profile": "prod-profile",
      "ignore": true,
      "regions": ["eu-west-1", "eu-west-3"]
    }
  },
  "naming_templates": {
    "std_env": {
      "regex": "^[^-]*-(.*)-([^-]*)$",
      "env": 1,
      "inst": 2
    }
  }
}
```

This example configuration file demonstrates key features:

-   `"default_env": "noenv"`: Specifies that if an instance name doesn't match any naming template or if a template doesn't explicitly define an environment, it will be assigned to the "noenv" environment.
-   `"default_regions": ["eu-west-3"]`: Sets "eu-west-3" as the default AWS region to scan for instances if an account definition doesn't specify its own regions.
-   `"default_templates": ["std_env"]`: Applies the naming template named "std_env" by default to accounts that don't list specific templates.
-   `"accounts"`: Defines two AWS accounts:
    -   `"dev"`: Uses the AWS profile named "dev-profile". It will use the `default_regions` and `default_templates`.
    -   `"prod"`: Uses the "prod-profile", and overrides the default regions, scanning "eu-west-1" and "eu-west-3". The `"ignore": true` flag means that instances from this account won't be processed during a full database refresh (`qss --refresh`) unless the "prod" account is explicitly specified (e.g., `qss --refresh prod`). This is useful for accounts you might not always want to scan or for which you have restricted/temporary access.
-   `"naming_templates"`: Defines how instance names are parsed to obtain their environment name and instance short name. Only one template, named `std_env`, is declared in this example:
    -   `"std_env"`: This template uses the regular expression `^[^-]*-(.*)-([^-]*)$`.
        -   `"env": 1`: The first capturing group `(.*)` of the regex will be used as the environment name.
        -   `"inst": 2`: The second capturing group `([^-]*)$` will be used as the instance short name.
        For example, for an instance named `myproject-staging-app01`, this template would extract `staging` as the environment and `app01` as the short name.

This configuration allows QSS to understand your AWS account structure, how to authenticate, which regions to target, and how to interpret your instance naming conventions to build its local database for quick connections.
A comprehensive description of all configuration options is available below in the "Configuration Options" section.

After creating or editing the configuration file, you'll typically want to refresh the QSS database by running the refresh command:

```bash
qss --refresh
```
This performs a full refresh of the database based on your configuration.
You can then verify that the database content matches your expectations:

```bash
qss --list --verbose
```
This command will print a table showing your instances, grouped by environment and account, including the full instance names.

### Configuration Options

All configuration options are defined within the `~/.qss.json` file.

-   **`default_env`**
    -   Type: `string`
    -   Default: `"."`
    -   Description: The catch-all environment name assigned to instances if their `Name` tag doesn't match any regex in the applied naming templates. In such cases, the instance's full `Name` tag is used as its short name. If `default_env` is set to an empty string (`""`), these instances will not be indexed in the local database. `default_env` is also used as the default environment name for templates that don't explicitly yield an environment name.

-   **`default_regions`**
    -   Type: `array` of `string`
    -   Default: `["us-east-1"]`
    -   Description: A list of AWS region names to scan for instances if an account definition in the `accounts` section does not specify its own `regions` list.

-   **`default_templates`**
    -   Type: `array` of `string`
    -   Default: `[]` (empty array)
    -   Description: A list of naming template names (which must be defined under the `naming_templates` section) to apply by default if an account definition in the `accounts` section does not specify its own `templates` list. Templates are tried in the order they are listed; the first one that matches an instance name is used.

-   **`default_account`**
    -   Type: `string` (must be lowercase)
    -   Default: Not set (undefined)
    -   Description: The name of an AWS account (which must be defined as a key in the `accounts` section) to use by default if no account is specified on the command line. If set, the `<account>` argument has to be omitted from QSS commands. This can be overridden using the `--account` or `-a` command-line option.

-   **`group_by_region`**
    -   Type: `boolean`
    -   Default: `false`
    -   Description: Determines the primary grouping for output in `--list` and `--info` commands. If `true`, instances are grouped by Account/Region/Environment. If `false` (default), they are grouped by Account/Environment/Region. This global default can be overridden per command execution using the `--group_by_region` (`-R`) or `--group_by_env` (`-E`) command-line options.

-   **`db_filepath`**
    -   Type: `string`
    -   Default: `~/.qss.db`
    -   Description: The file path for the local QSS instance database. Standard environment variables in the path (e.g., `$HOME`, `${HOME}` on Unix-like systems, or `%VAR%` on Windows) will be expanded. The `~` character is also expanded to the user's home directory.

-   **`connect_command`**
    -   Type: `array` of `string`
    -   Default: `["aws", "--profile", "<%= profile %>", "--region", "<%= region %>", "ssm", "start-session", "--target", "<%= instance_id %>"]`
    -   Description: An array of strings representing the command and its arguments used to establish the connection to the target instance. Each string in the array is treated as an ERB (Embedded Ruby) template.
        The following variables are available for use within these ERB templates:
        -   `account`: The QSS account name (e.g., `dev`).
        -   `profile`: The AWS CLI profile name configured for the account.
        -   `region`: The AWS region of the instance (e.g., `eu-west-3`).
        -   `az`: The full availability zone of the instance (e.g., `eu-west-3a`).
        -   `env`: The QSS environment name (e.g., `staging`).
        -   `short_name`: The QSS instance short name (e.g., `app-server`).
        -   `instance_id`: The EC2 instance ID (e.g., `i-0123456789abcdef0`).
        -   `instance_name`: The value of the EC2 instance's "Name" tag.
        -   `ec2_instance`: The complete `Aws::EC2::Instance` object for the target instance, as returned by the AWS SDK for Ruby. This provides access to all instance attributes like IP addresses, VPC ID, security groups, tags, etc. (e.g., `<%= ec2_instance.public_ip_address %>`, `<%= ec2_instance.tags.find { |t| t.key == 'Owner' }&.value %>`). Refer to the _AWS SDK for Ruby V3 documentation for `Aws::EC2::Instance`_ for a full list of available attributes and methods.

-   **`connect_env_vars`**
    -   Type: `object` (string keys, string values)
    -   Default: `{}` (empty object)
    -   Description: An object where keys are environment variable names and values are their corresponding string values. These environment variables will be set in the environment before executing the `connect_command`. The values are also ERB templates, with the same set of variables available as for `connect_command`.

-   **`accounts`**
    -   Type: `object`
    -   Description: An object where each key is a custom account name (must be lowercase, e.g., `"dev"`, `"production"`) and the value is an object defining that account's specific properties.
    -   Each account object can have the following sub-properties:
        -   **`aws_profile`** (Type: `string`, *Required*): The AWS CLI profile name to use for authenticating API calls for this account. If an empty string is provided, no profile will be explicitly used (useful if QSS is run from an EC2 instance with an appropriate IAM role).
        -   **`ignore`** (Type: `boolean`, Optional): If set to `true`, instances from this account are not processed during a full database refresh (`qss --refresh`) and are excluded from general listings (`qss --list`, `qss --info`), unless the account is explicitly specified in the command (e.g., `qss --refresh prod`, `qss --list prod`). This is useful for accounts with restricted or infrequent access, or to speed up general operations. Defaults to `false` if not specified.
        -   **`regions`** (Type: `array` of `string`, Optional): A list of AWS region names to scan specifically for this account. If provided, this list overrides the global `default_regions` for this account.
        -   **`templates`** (Type: `array` of `string`, Optional): A list of naming template names (which must be defined under the `naming_templates` section) to apply specifically to instances within this account. If provided, this list overrides the global `default_templates` for this account. Templates are tried in the order they are listed.

-   **`naming_templates`**
    -   Type: `object`
    -   Description: An object where each key is a custom template name (e.g., `"standard_format"`, `"legacy_apps"`) and the value is an object defining the template's parsing logic for EC2 instance `Name` tags.
    -   Each template object can have the following sub-properties:
        -   **`regex`** (Type: `string`, *Required*): The regular expression (Ruby syntax) used to parse the EC2 instance's "Name" tag. It can use capturing groups to extract parts of the name.
        -   **`env`** (Type: `integer` or `string`, Optional): Defines how to extract the environment name from the instance's "Name" tag using the `regex`.
            -   If an `integer`, it's the 1-based index of the capturing group in the `regex` match (e.g., `1` for the first group).
            -   If a `string`, it's an ERB template. The following variables are available: `match` (the Ruby `MatchData` object from the regex), `account` (QSS account name), `region` (AWS region), `az` (availability zone), `inst_name` (the full "Name" tag of the instance). Example: `"<%= match[1] %>-<%= region %>"`.
            -   If this property is not provided, the value of `default_env` will be used. If the *final* environment name (after considering `default_env`) is an empty string, the instance will not be indexed.
        -   **`inst`** (Type: `integer` or `string`, Optional): Defines how to extract the instance short name from the instance's "Name" tag using the `regex`.
            -   If an `integer`, it's the 1-based index of the capturing group.
            -   If a `string`, it's an ERB template (same variables as for `env`).
            -   If this property is not provided, the instance's full "Name" tag (`inst_name`) will be used as its short name. If the *final* short name is an empty string, the instance will not be indexed.

## Usage

### Connect to an Instance

Connect to an EC2 instance via SSM Session Manager. QSS will resolve the instance ID based on the provided account, environment, and short name.

```bash
# Connect to instance 'app-server' in environment 'staging' of account 'dev'
qss dev staging app-server
```
If multiple running instances match the provided criteria (e.g., if several instances have the same `Name` tag , or if a short name maps to multiple full instance names), QSS will list the available running instances and prompt you to select which one you want to connect to. If only one running instance is found but other non-running (e.g., stopped) instances also matched, QSS will inform you and ask for confirmation before connecting.

### Refresh the Instance Database

Update the local QSS database with the latest instance information from AWS.

```bash
# Refresh all non-ignored accounts and their configured regions
qss --refresh

# Refresh only the 'dev' account
qss --refresh dev

# Refresh only the 'staging' environment within the 'dev' account
qss --refresh dev staging
```

### List Instances

List instances from the local QSS database.

```bash
# List all instances from all non-ignored accounts
qss --list

# List instances in the 'dev' account
qss --list dev

# List instances in the 'staging' environment of the 'dev' account
qss --list dev staging

# List instances, grouped by region first, then environment
qss --list --group_by_region

# List instances with verbose output (shows full instance names)
qss --list --verbose
```

### Display Detailed Instance Information

Fetch and display detailed real-time information about EC2 instances from AWS.

```bash
# Show information for all instances in all non-ignored accounts
qss --info

# Show information for instances in the 'staging' environment of the 'dev' account
qss --info dev staging

# Show detailed information for a specific instance 'app-server'
qss --info dev staging app-server

# Show verbose information (includes more details like AMI, IPs, tags, etc.)
# Tags are only displayed when an instance short name is specified
qss --info --verbose dev staging app-server
```

### Using a Default Account

If a `default_account` is defined in your `~/.qss.json` configuration file:

```bash
# Connect to 'app-server' in 'staging' environment using the default account
qss staging app-server

# List instances in the default account
qss --list

# You can still override the default and specify a different account
qss --account prod staging app-server

# To perform an operation on all accounts (e.g., a full refresh) when a
# default_account is set, you can explicitly pass an empty account name
# with the --account option:
qss --refresh --account ""
```

### All Options

This section details all available command-line options for `qss`.

-   **`-r, --refresh`** `[<account> [<env name>]]`
    -   Refreshes the local instance database.
    -   Without arguments, refreshes all non-ignored accounts.
    -   With an account name, refreshes only that account.
    -   With an account and environment name, refreshes only that specific environment within the account.

-   **`-l, --list`** `[<account> [<env name>]]`
    -   Lists the content of the local instance database.
    -   Can be filtered by account and/or environment.
    -   Use with `--verbose` to see full instance names.

-   **`-i, --info`** `[<account> [<env name> [<instance short name>]]]`
    -   Displays detailed information about instances by querying AWS EC2 in real-time.
    -   Can be filtered by account, environment, and/or instance short name.
    -   Use with `--verbose` for extended details (instance type, AMI, IPs, tags, etc.).

-   **`--install-completion`**
    -   Installs the bash completion script for `qss`.
    -   The script will guide you through the installation process.

-   **`-R, --group_by_region`**
    -   Used with `--list` or `--info` to group instances by Account/Region/Environment.
    -   Overrides the `group_by_region` setting from the configuration file for this execution.

-   **`-E, --group_by_env`**
    -   Used with `--list` or `--info` to group instances by Account/Environment/Region (default behavior if `group_by_region` is not `true` in the configuration).
    -   Overrides the `group_by_region` setting from the configuration file for this execution, forcing it to `false`.

-   **`-a, --account <account_name>`**
    -   Specifies the account name to use for the command.
    -   Overrides the `default_account` setting from the configuration file.
    -   Can be used with an empty string (e.g., `-a ""`, `-a ''`, or `--account=`) to indicate "no specific account" when used with commands like `--refresh` if a `default_account` is configured but a global operation is desired.

-   **`-v, --verbose`**
    -   Enables verbose mode, displaying more details during execution and in the output of `--list` and `--info` commands.
    -   Repeating the option like `-vv` increases the verbosity of the script, adding messages relating to the internal functioning of the script.

-   **`-y, --yes`**
    -   Automatically answers "yes" to all confirmation prompts (e.g., when connecting to an instance with resolved ambiguities, or during completion script installation).

-   **`-h, --help`**
    -   Displays the help message and exits.

-   **`-c, --config <filepath>`**
    -   Specifies the path to the JSON configuration file to use.
    -   Default: `~/.qss.json`.

## Advanced Usage

### Custom Connection Commands

You can customize the command used to connect to instances by modifying the `connect_command` array in your `~/.qss.json` configuration file. This allows for advanced scenarios, such as using SSH instead of the default SSM Session Manager.

The `ec2_instance` ERB variable gives you access to the full `Aws::EC2::Instance` object, enabling you to retrieve any instance attribute, like its public or private IP address, for your custom command.

**Example: Using SSH instead of SSM**

This configuration snippet shows how to set up QSS to initiate an SSH connection.

```json
{
  "connect_command": [
    "ssh",
    "-i",
    "~/.ssh/my_key",
    "ec2-user@<%= ec2_instance.public_ip_address %>"
  ]
}
```

### Example Using Custom Environment Variables and a Wrapper Script

You can set environment variables before the `connect_command` executes using the `connect_env_vars` configuration. This is particularly useful when your `connect_command` is a custom script that relies on these variables.

The following example demonstrates how to pass AWS profile, region, and instance ID to a wrapper script named `ssm_session_reconnect.sh`. This script implements a loop to automatically offer reconnection if the SSM session is terminated.

**Configuration in `~/.qss.json`:**
```json
{
  "connect_command": [
    "ssm_session_reconnect.sh"
  ],
  "connect_env_vars": {
    "QSS_AWS_PROFILE": "<%= profile %>",
    "QSS_AWS_REGION": "<%= region %>",
    "QSS_INSTANCE_ID": "<%= instance_id %>"
  }
}
```

**Example `ssm_session_reconnect.sh` script:**
```sh
#! /bin/sh

# ssm_session_reconnect.sh
# This script attempts to reconnect to an SSM session if it closes.

# Construct the AWS command
AWS_CMD_PARTS="aws"
if [ -n "$QSS_AWS_PROFILE" ]; then
  AWS_CMD_PARTS="$AWS_CMD_PARTS --profile \"$QSS_AWS_PROFILE\""
fi
AWS_CMD_PARTS="$AWS_CMD_PARTS --region \"$QSS_AWS_REGION\" ssm start-session --target \"$QSS_INSTANCE_ID\""

while true; do
  echo "Attempting to connect..."
  # Execute the command. Using eval to correctly interpret quotes in AWS_CMD_PARTS
  eval $AWS_CMD_PARTS

  echo "Connection closed at $(date '+%Y-%m-%d %H:%M:%S')"
  printf "Press Enter to reconnect, or Ctrl-C to exit"
  read REPLY_UNUSED
done
```
Make sure this script (`ssm_session_reconnect.sh`) is executable and located in a directory included in your system's `PATH`, or provide the full path to it in the `connect_command`.

## License

This project is licensed under the MIT License - see the LICENSE file for details.
