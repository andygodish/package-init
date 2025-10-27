# Package: Zarf Init

Deploy the [zarf init](https://github.com/zarf-dev/zarf) package tailored to your environment.

## UDS CLI & Maru

Maru tasks are used to pull in the upstream K3D package via the `uds run` alias built into the UDS CLI tool defined in the tasks.yaml file.

## Variable Overrides via Config Files

[Zarf Docs](https://docs.zarf.dev/ref/config-files/#config-file-examples)

Create a default config file in your pwd by running `zarf dev generate-config`

### Deploying an Alternate Config File

Problem: You want to deploy the zarf init package with an alternate config file, e.g. `configs/no-gitea.toml`:

The tasks.yaml declares CONFIG_FILE with the default zarf-config.toml, and Maru populates all variables from that block before it expands cmd strings. That means the value you set on the shell (CONFIG_FILE=./configs/no-gitea.toml uds run deploy) gets overwritten by the default before the task runs, so the deploy step still sees zarf-config.toml as the config.
The deploy action simply forwards whatever is in CONFIG_FILE via ZARF_CONFIG="$CONFIG_FILE" (tasks.yaml:43-45). Because the value never changed, Zarf reports that it is using the root config file.
To run with your alternate config, override the task variable instead of the shell env:

The key is the --set flag to uds run:

```bash
uds run deploy --set CONFIG_FILE=./configs/no-gitea.toml
```

That keeps everything else the same while pointing Zarf at configs/no-gitea.toml. If you want shell environment overrides to work in the future, you could drop the CONFIG_FILE entry from the variables list or add logic in the script to respect an externally provided value, but using --set is the intended mechanism here.

Next step: retry the deploy with the --set flag and confirm Zarf now reads the expected config.
