Deployment hooks
================

Deployment hooks enable services to program and gate their deployments. This is
often used for testing but can also be used for things like deploying static
assets to a CDN or building other types of automation into a deployment.

Hooks are Dockerfiles meaning hook authors can choose any language or runtime.

Hooks come in two forms:

1. Dockerfiles that live in the `.porter/hooks/` directory
1. git repos containing one or more Dockerfiles. These are called plugins.

All hooks are optional and will only be called if they exist. If they exist they
must exit with a code of 0 to continue the deployment. Any non-zero exit code is
considered a failure and will halt whatever command was called that caused the
hook to be called.

There are currently 9 hooks available. 8 of them are used during the 4 build
phases (pre and post), the other is used to customize EC2 initialization.

The `pre` and `post` hooks are tied to the `porter build ...` command names, not
the underlying mechanisms of provisioning, promoting, etc.

The hooks
---------

- [pre-pack](hooks/pre-pack.md)
- [post-pack](hooks/post-pack.md)
- [pre-provision](hooks/pre-provision.md)
- [post-provision](hooks/post-provision.md)
- [pre-promote](hooks/pre-promote.md)
- [post-promote](hooks/post-promote.md)
- [pre-prune](hooks/pre-prune.md)
- [post-prune](hooks/post-prune.md)
- [ec2-bootstrap](hooks/ec2-bootstrap.md)

pre hook vs. post hook
----------------------

When to use a pre hook vs a post hook is determined by what build command you
want the hook to operate as part of.

For example, `post-provision` and `pre-promote` aren't very different. The state
of the deployment hasn't changed and all of the same variables are available.
The only difference is that `post-provision` is called during `porter build
provision`, while `pre-promote` is called during `porter build promote`.

That difference is probably only meaningful if these commands are run on
different boxes, with different environment variables, or are otherwise
logically separate from the caller's perspective.

The recommendation is to use pre hooks and then add or change to post hooks as
hooks are better understood or requirements dictate.

Hook environment
----------------

Each hook's Docker context is the directory containing the hook.

The repo's root is volume mapped to `/repo_root` so hooks can access and mutate
the contents of a repo.

Environment variables are injected as they are available.

### Standard environment variables

These are available to all hooks

```
PORTER_SERVICE_NAME
PORTER_SERVICE_VERSION (git sha)
DOCKER_ENV_FILE
HAPROXY_STATS_USERNAME
HAPROXY_STATS_PASSWORD
HAPROXY_STATS_URL
```

### Custom environment variables

Additional environment variables can be injected by prefixing them with
`PORTER_`. The environment in which porter is called is different that the
environment that a container may have. It is necessary to prefix the environment
in which porter is called so porter knows which environment variables it should
inject into the container. Injecting all environment variables that are
available to porter (such as `$HOME` and `$PATH`) would cause problems if passed
into the container.

As an example consider calling `PORTER_FOO=bar porter create-stack -e dev`. The
prefix `PORTER_` is stripped by porter and injected into the container hook as
`FOO`. This enables code run in docker containers to be configured outside of
porter.

Plugins
-------

Single hooks are hardly sufficient for non-trivial projects. Porter enables
hooks to be developed independently and referenced by porter projects in
`.porter/config`.

Plugins are porter hooks that live in a separate repo.

All plugins defined in the config are run after any single hooks that live in a
repo. For example if we also define a `.porter/hooks/ec2-bootstrap` and use the
config above, porter-contrib-datadog will run after
`.porter/hooks/ec2-bootstrap`. Services shouldn't rely on this ordering,
however.

Plugins can run as part of more than one hook. By default porter looks for a
`dockerfile` value in the hook config. If this isn't found it defaults to
`Dockerfile`. Here is an example config for a made up plugin called
porter-contrib-foo that runs for both `pre_pack` and `post_pack`, and a
porter-contrib-bar that contains a single Dockerfile at its root (to illustrate
the behavior).

```yaml
hooks:
  pre_pack:
  - repo: git@github.com:adobe-platform/porter-contrib-foo.git
    ref: v1.0.0
    dockerfile: pre-pack
  post_pack:
  - repo: git@github.com:adobe-platform/porter-contrib-foo.git
    ref: v1.0.0
    dockerfile: post-pack
  - repo: git@github.com:adobe-platform/porter-contrib-bar.git
    ref: v1.0.0
    # This is the default if undefined
    # dockerfile: Dockerfile
```

`pre-pack` and `post-pack` are Dockerfiles found in the root directory of the
porter-contrib-foo repo. There's no restriction on the name or placement of
these Dockerfiles (i.e. `pre-pack` could also have been `some_dir/Dockerfile`)
meaning a single repo could contain many hooks.
