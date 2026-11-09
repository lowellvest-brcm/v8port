[![Automatic version updates](https://github.com/ZOSOpenTools/v8port/actions/workflows/bump.yml/badge.svg)](https://github.com/ZOSOpenTools/v8port/actions/workflows/bump.yml)

# v8 port

A port of Google V8 JavaScript engine to z/OS Open Tools project

The upstream for v8 is [here.](https://v8.dev)

## Developer Notes

### Prerequisite

The manual workflow to build v8port requires the following:

* python
    - tested with python 3.13 (IBM Open Enterprise SDK for Python 3.13)
* ninja from z/OS Open Tools [ninjaport](https://github.com/ZOSOpenTools/ninjaport)
* gn from zos open tools [gnport](https://github.com/ZOSOpenTools/gnport)
* IBM Open XL C/C++ 2.2 for z/OS, with `ibm-clang` and `ibm-clang++` on the `PATH`
    - The V8 `build` repository invokes the compiler as `ibm-clang`/`ibm-clang++`, and
      the patches here assume Open XL 2.2 (z/OS target level `zosv2r5`).
    - The older "C/C++ for Open Enterprise Languages on z/OS 2.0" (clang 14, `clang++`
      only) is not supported: it lacks those driver names and cannot compile this
      V8 revision (for example it rejects C++20 code that omits `typename`).
    ```
    $ ibm-clang++ --version
    IBM Open XL C/C++ 2.2 for z/OS, version 2.2.0.0, clang version 21.1.1 (build 8a7b88c)
    Target: s390-ibm-zos-stdlink
    Thread model: posix
    ```
* depot_tools port  zos open tools depot_toolsport
* zoslib
    - The manual workflow provides instructions on how to install this. It uses the same
      compiler as above.



## z/OS Open Tools Workflow 

This is the workflow for building v8 using the z/OS Open Tools framework.

1. install depot_tools 
    - `$ zopen install depot_tools`
2. Currently v8port is still in developer form
    - `$ cd ${HOME}/zopen/dev`
    - `$ git clone git@github.com:ZOSOpenTools/v8port.git`
3. The build process will use git via ssh.  If you have a passphrase
   for your ssh key, do the following:
   - `$ eval \`ssh-agent\``  
   - `$ ssh-add ${HOME}/.ssh/some_private_ssh_key`
4. Start the build process
    - `$ cd ${HOME}/zopen/dev/v8port`
    - `$ rm -rf ~/.local v8base install ${HOME}/zopen/usr/local/zopen/v8 venv log; zopen build > out.log 2>&1`

It will create a venv in `./venv` of the v8port directory.
The v8port `buildenv` file will setup config the environment to 
fetch the v8 source code. Afterwards, source this venv via
`. ./venv/bin/activate`
to work on the repo.

The build fails (and nothing is installed) if the fetch, the dependency sync, `gn`
or `ninja` fails. Only the build products are installed (`d8`, the shared libraries,
the V8 headers), not the source tree.

This can be done in a convienece script via

```
$ . ./setenv.sh
```

or if you want to preserve the hidden `v8base/.gclient` and `v8base/.gclient_entries`

```
$ . ./setenv.sh -resume
```



### Environment and troubleshooting

Things that stop `zopen build` early and the symptom they produce in `out.log`:

* **TLS errors from `curl`, `git` or `gclient`** (`unable to get local issuer certificate`): the
  system CA bundle is missing or too old. Point `SSL_CERT_FILE`, `CURL_CA_BUNDLE`,
  `REQUESTS_CA_BUNDLE` and `GIT_SSL_CAINFO` (and `git config --global http.sslCAInfo`) at a
  current CA bundle.
* **`pip` cannot download packages** (`Network is unreachable` to `files.pythonhosted.org`):
  use a mirror that is reachable from the system, for example via `PIP_INDEX_URL`.
* **`ModuleNotFoundError` from `gclient.py`**: the packages in `requirements.txt` were not
  installed in `venv`. `httplib2` is pinned to 0.22.0 because depot_tools imports
  `httplib2.socks`, which later releases removed.
* **`CEE3561S External function ... was not found in DLL libzoslib.so`** when `torque` or
  `mksnapshot` runs: the `libzoslib.so` built in `out/zos_s390x.release` must come first on
  `LIBPATH` (`setenv.sh` does this).
* **Not enough space**: the source tree and dependencies under `v8base` are about 6 GB, and
  `out/` needs more. Allow roughly 10 GB for the file system holding `v8base`.
