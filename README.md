# Project B

This project imports data from [_Project A_](https://github.com/sisp/dvc-import-auth-example/tree/project-a) via DVC. It serves as a minimal reproducible example for investigating why DVC cannot pull imported data from the private DVC remote although access credentials are configured.

## How this project/branch was created

1.  Initialize DVC:

    ```shell-session
    $ dvc init
    Initialized DVC repository.

    You can now commit the changes to git.

    +---------------------------------------------------------------------+
    |                                                                     |
    |        DVC has enabled anonymous aggregate usage analytics.         |
    |     Read the analytics documentation (and how to opt-out) here:     |
    |             <https://dvc.org/doc/user-guide/analytics>              |
    |                                                                     |
    +---------------------------------------------------------------------+

    What's next?
    ------------
    - Check out the documentation: <https://dvc.org/doc>
    - Get help and share ideas: <https://dvc.org/chat>
    - Star us on GitHub: <https://github.com/iterative/dvc>
    ```

1.  Configure the DVC remote that contains the data imported from _Project A_:

    ```sh
    dvc remote add gitlab.com:40913719 https://gitlab.com/api/v4/projects/40913719/packages/generic/dvc-data
    dvc remote modify gitlab.com:40913719 method PUT
    dvc remote modify gitlab.com:40913719 auth custom
    ```

    > See in the next section why it appears that this step is necessary.

1.  Import the data from _Project A_:

    ```shell-session
    $ dvc import --rev project-a -o project-a-data.txt --no-download git@github.com:sisp/dvc-import-auth-example.git data.txt
    Importing 'data.txt (git@github.com:sisp/dvc-import-auth-example.git)' -> 'project-a-data.txt'
    ```

## How to reproduce the problem

In the DVC documentation on `dvc import` about [chained imports](https://dvc.org/doc/command-reference/import#example-chained-imports), it says at the very end:

> The [default DVC remotes](https://dvc.org/doc/command-reference/remote/default) for all repos in the import chain must also be accessible (repo C needs to have all the appropriate credentials).

This sounds to me as if it should be possible to configure the access credentials for the DVC remote (in this case, it's a GitLab deploy token with scope `read_package_registry`) of the imported data as follows:

```shell-session
$ dvc remote modify --local gitlab.com:40913719 custom_auth_header 'DEPLOY-TOKEN'
ERROR: configuration error - config file error: remote 'gitlab.com:40913719' doesn't exist.
dvc remote modify --local gitlab.com:40913719 password XkHJH5DiXpvsgwpy4dMt
ERROR: configuration error - config file error: remote 'gitlab.com:40913719' doesn't exist.
```

But DVC doesn't recognize the DVC remote from the imported Git repository. So I added the DVC remote in this project again:

```sh
dvc remote add gitlab.com:40913719 https://gitlab.com/api/v4/projects/40913719/packages/generic/dvc-data
dvc remote modify gitlab.com:40913719 method PUT
dvc remote modify gitlab.com:40913719 auth custom
dvc remote modify --local gitlab.com:40913719 custom_auth_header 'DEPLOY-TOKEN'
dvc remote modify --local gitlab.com:40913719 password XkHJH5DiXpvsgwpy4dMt
```

Despite the configured credentials, pulling the imported data from the DVC remote fails:

```shell-session
$ dvc pull
ERROR: configuration error - HTTP 'custom' authentication require both 'custom_auth_header' and 'password'
ERROR: HTTP 'custom' authentication require both 'custom_auth_header' and 'password'
Learn more about configuration settings at <https://man.dvc.org/remote/modify>.
```

Here is the full stack trace:

<details>

```shell-session
$ dvc pull -v
2022-11-09 12:19:41,626 DEBUG: Creating external repo git@github.com:sisp/dvc-import-auth-example.git@f3496fd5bcb994aab5eb43e37aebb31479d6f7dc
2022-11-09 12:19:41,626 DEBUG: erepo: git clone 'git@github.com:sisp/dvc-import-auth-example.git' to a temporary dir
2022-11-09 12:19:46,917 DEBUG: Checking if stage '/data.txt' is in 'dvc.yaml'
2022-11-09 12:19:47,114 ERROR: configuration error - HTTP 'custom' authentication require both 'custom_auth_header' and 'password'
------------------------------------------------------------
Traceback (most recent call last):
  File ".../site-packages/dvc/cli/__init__.py", line 185, in main
    ret = cmd.do_run()
  File ".../site-packages/dvc/cli/command.py", line 22, in do_run
    return self.run()
  File ".../site-packages/dvc/commands/data_sync.py", line 31, in run
    stats = self.repo.pull(
  File ".../site-packages/dvc/repo/__init__.py", line 48, in wrapper
    return f(repo, *args, **kwargs)
  File ".../site-packages/dvc/repo/pull.py", line 34, in pull
    processed_files_count = self.fetch(
  File ".../site-packages/dvc/repo/__init__.py", line 48, in wrapper
    return f(repo, *args, **kwargs)
  File ".../site-packages/dvc/repo/fetch.py", line 83, in fetch
    d, f = _fetch(
  File ".../site-packages/dvc/repo/fetch.py", line 138, in _fetch
    used = repo.used_objs(
  File ".../site-packages/dvc/repo/__init__.py", line 431, in used_objs
    for odb, objs in self.index.used_objs(
  File ".../site-packages/dvc/repo/index.py", line 267, in used_objs
    for odb, objs in stage.get_used_objs(
  File ".../site-packages/dvc/stage/__init__.py", line 682, in get_used_objs
    for odb, objs in out.get_used_objs(*args, **kwargs).items():
  File ".../site-packages/dvc/output.py", line 1036, in get_used_objs
    return self.get_used_external(**kwargs)
  File ".../site-packages/dvc/output.py", line 1091, in get_used_external
    return dep.get_used_objs(**kwargs)
  File ".../site-packages/dvc/dependency/repo.py", line 99, in get_used_objs
    used, _, _ = self._get_used_and_obj(**kwargs)
  File ".../site-packages/dvc/dependency/repo.py", line 128, in _get_used_and_obj
    odb = repo.cloud.get_remote_odb()
  File ".../site-packages/dvc/data_cloud.py", line 90, in get_remote_odb
    remote = self.get_remote(name=name, command=command)
  File ".../site-packages/dvc/data_cloud.py", line 66, in get_remote
    fs = cls(**config)
  File ".../site-packages/dvc_objects/fs/base.py", line 80, in __init__
    self.fs_args.update(self._prepare_credentials(**kwargs))
  File ".../site-packages/dvc_http/__init__.py", line 67, in _prepare_credentials
    raise ConfigError(
dvc_objects.fs.errors.ConfigError: HTTP 'custom' authentication require both 'custom_auth_header' and 'password'
------------------------------------------------------------
2022-11-09 12:19:47,115 ERROR: HTTP 'custom' authentication require both 'custom_auth_header' and 'password'
Learn more about configuration settings at <https://man.dvc.org/remote/modify>.
------------------------------------------------------------
Traceback (most recent call last):
  File ".../site-packages/dvc/cli/__init__.py", line 185, in main
    ret = cmd.do_run()
  File ".../site-packages/dvc/cli/command.py", line 22, in do_run
    return self.run()
  File ".../site-packages/dvc/commands/data_sync.py", line 31, in run
    stats = self.repo.pull(
  File ".../site-packages/dvc/repo/__init__.py", line 48, in wrapper
    return f(repo, *args, **kwargs)
  File ".../site-packages/dvc/repo/pull.py", line 34, in pull
    processed_files_count = self.fetch(
  File ".../site-packages/dvc/repo/__init__.py", line 48, in wrapper
    return f(repo, *args, **kwargs)
  File ".../site-packages/dvc/repo/fetch.py", line 83, in fetch
    d, f = _fetch(
  File ".../site-packages/dvc/repo/fetch.py", line 138, in _fetch
    used = repo.used_objs(
  File ".../site-packages/dvc/repo/__init__.py", line 431, in used_objs
    for odb, objs in self.index.used_objs(
  File ".../site-packages/dvc/repo/index.py", line 267, in used_objs
    for odb, objs in stage.get_used_objs(
  File ".../site-packages/dvc/stage/__init__.py", line 682, in get_used_objs
    for odb, objs in out.get_used_objs(*args, **kwargs).items():
  File ".../site-packages/dvc/output.py", line 1036, in get_used_objs
    return self.get_used_external(**kwargs)
  File ".../site-packages/dvc/output.py", line 1091, in get_used_external
    return dep.get_used_objs(**kwargs)
  File ".../site-packages/dvc/dependency/repo.py", line 99, in get_used_objs
    used, _, _ = self._get_used_and_obj(**kwargs)
  File ".../site-packages/dvc/dependency/repo.py", line 128, in _get_used_and_obj
    odb = repo.cloud.get_remote_odb()
  File ".../site-packages/dvc/data_cloud.py", line 90, in get_remote_odb
    remote = self.get_remote(name=name, command=command)
  File ".../site-packages/dvc/data_cloud.py", line 66, in get_remote
    fs = cls(**config)
  File ".../site-packages/dvc_objects/fs/base.py", line 80, in __init__
    self.fs_args.update(self._prepare_credentials(**kwargs))
  File ".../site-packages/dvc_http/__init__.py", line 67, in _prepare_credentials
    raise ConfigError(
dvc_objects.fs.errors.ConfigError: HTTP 'custom' authentication require both 'custom_auth_header' and 'password'
------------------------------------------------------------
2022-11-09 12:19:47,118 DEBUG: Analytics is enabled.
2022-11-09 12:19:47,226 DEBUG: Trying to spawn '['daemon', '-q', 'analytics', '/tmp/tmpz2r77ma7']'
2022-11-09 12:19:47,230 DEBUG: Spawned '['daemon', '-q', 'analytics', '/tmp/tmpz2r77ma7']
```

</details>

And DVC debug information:

```shell-session
$ dvc doctor
DVC version: 2.34.0 (pip)
---------------------------------
Platform: Python 3.9.13 on Linux-5.13.0-48-generic-x86_64-with-glibc2.31
Subprojects:
	dvc_data = 0.25.3
	dvc_objects = 0.12.2
	dvc_render = 0.0.12
	dvc_task = 0.1.4
	dvclive = 1.0.1
	scmrepo = 0.1.3
Supports:
	http (aiohttp = 3.8.3, aiohttp-retry = 2.8.3),
	https (aiohttp = 3.8.3, aiohttp-retry = 2.8.3)
Cache types: <https://error.dvc.org/no-dvc-cache>
Caches: local
Remotes: https
Workspace directory: ext4 on /dev/mapper/vgubuntu-root
Repo: dvc, git
```