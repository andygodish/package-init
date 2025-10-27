# Components in Zarf Configuration Files

All upstream package components marked with `required: true` are installed by default at deploy time and cannot be disabled. Optional components can be enabled or disabled via the `components` field in the `[package.deploy]` section of the Zarf configuration file (e.g. `zarf-config.toml`).

The Zarf Init package includes the following components:

```yaml
kind: ZarfInitConfig
metadata:
  name: init
  description: Used to establish a new Zarf cluster
  allowNamespaceOverride: false

components:
  - name: k3s
    import:
      path: packages/distros/k3s

  # This package moves the injector & registries binaries
  - name: zarf-injector
    required: true
  - name: zarf-seed-registry
    required: true
  - name: zarf-registry
    required: true
  - name: zarf-agent
    required: true

  - name: git-server
```

Only the `git-server` component is optional. It can be added by using the components field in your zarf-config.toml like so:

```toml
components = 'git-server'
```

That will result in all 5 components being installed. Run `zarf package list` to see the full list of components available in the deployed package:

```bash
     Package | Namespace Override | Version | Components                                                            
     init    |                    | v0.64.0 | [zarf-injector zarf-seed-registry zarf-registry zarf-agent git-server]
     uds-k3d |                    | 0.18.0  | [destroy-cluster create-cluster uds-dev-stack] 
```

The problem with the `toml` configuration above is that when you go to remove the package, it will only target the `git-server` component for removal, leaving the other 4 components behind. Furthermore, the `zarf package remove` command always appears to target the local zarf-config.toml file, regardless of which config file was used at deploy time. There does not appear to be a way to specify an alternate config file at removal time.

```zsh
package-init git:(main) ✗ uds z p remove init --confirm
2025-10-27 16:25:02 INF using config file location=/Users/andyg/src/github.com/andygodish/package-init/zarf-config.toml
```

Therefore, if you wanted to remove everything cleanly, you would need to ensure that your zarf-config.toml file included all 5 components at removal time, like so:

```toml
components = 'zarf-injector,zarf-seed-registry,zarf-registry,zarf-agent,git-server'
```
