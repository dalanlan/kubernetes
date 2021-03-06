# .kubeconfig files
In order to easily switch between multiple clusters, a .kubeconfig file was defined.  This file contains a series of authentication mechanisms and cluster connection information associated with nicknames.  It also introduces the concept of a tuple of authentication information (user) and cluster connection information called a context that is also associated with a nickname.

Multiple files are .kubeconfig files are allowed.  At runtime they are loaded and merged together along with override options specified from the command line (see rules below).

## Related discussion
https://github.com/GoogleCloudPlatform/kubernetes/issues/1755

## Example .kubeconfig file
```
apiVersion: v1
clusters:
- cluster:
    api-version: v1beta1
    server: http://cow.org:8080
  name: cow-cluster
- cluster:
    certificate-authority: path/to/my/cafile
    server: https://horse.org:4443
  name: horse-cluster
- cluster:
    insecure-skip-tls-verify: true
    server: https://pig.org:443
  name: pig-cluster
contexts:
- context:
    cluster: horse-cluster
    namespace: chisel-ns
    user: green-user
  name: federal-context
- context:
    cluster: pig-cluster
    namespace: saw-ns
    user: black-user
  name: queen-anne-context
current-context: federal-context
kind: Config
preferences:
  colors: true
users:
- name: black-user
  user:
    auth-path: path/to/my/existing/.kubernetes_auth_file
- name: blue-user
  user:
    token: blue-token
- name: green-user
  user:
    username: admin
    password: secret
    client-certificate: path/to/my/client/cert
    client-key: path/to/my/client/key
```

## .kubernetes_auth files

**WARNING**: merging auth from a mixture of kubernetes_auth file entries and .kubeconfig user entries is hard to debug and should be avoided. kubernetes_auth file support exists mostly for tests and is being deprecated.

The kubernetes_auth file is a legacy config file that can contain a mix of server and client auth info. It is supported in .kubeconfig via `auth-path` for a user:
```
users:
- name: black-user
  user:
    auth-path: path/to/my/existing/.kubernetes_auth_file
```

The entries in a file loaded via auth-path will be applied to both the user and cluster of the current context.

### Example .kubernetes_auth file
```
{
  "User": "admin",
  "Password": "secret",
  "CertFile": "/path/to/my/client/cert",
  "KeyFile": "/path/to/my/client/key",
  "CAFile": "/path/to/my/server/cafile",
  "BearerToken": "secrettoken",
  "Insecure": false
}
```
All entries are optional. `User`, `Password`, `CertFile`, `KeyFile`, and `BearerToken` are applied to the kubectl user. `CAFile` and `Insecure` apply to the cluster. Note that it is invalid to set both `CAFile` and `Insecure`, or both `BearerToken` and `User,Password` (see loading and merging rules below).

If the contents of the kubernetes_auth file conflict with entries in .kubeconfig, they are ignored. E.g, if the .kubeconfig cluster specifies a `certificate-authority`, and the user specifies an `auth-path` to a kubernetes_file that contains a `CAFile` entry, the former will be used and the latter ignored.

## Loading and merging rules
The rules for loading and merging the .kubeconfig files are straightforward, but there are a lot of them.  The final config is built in this order:
  1.  Get the kubeconfig  from disk.  This is done with the following hierarchy and merge rules:
      

      If the CommandLineLocation (the value of the `kubeconfig` command line option) is set, use this file only.  No merging.  Only one instance of this flag is allowed.


      Else, if EnvVarLocation (the value of $KUBECONFIG) is available, use it as a list of files that should be merged.  
      Merge files together based on the following rules.
      Empty filenames are ignored.  Files with non-deserializable content produced errors.
      The first file to set a particular value or map key wins and the value or map key is never changed.
      This means that the first file to set CurrentContext will have its context preserved.  It also means that if two files specify a "red-user", only values from the first file's red-user are used.  Even non-conflicting entries from the second file's "red-user" are discarded.


      Otherwise, use HomeDirectoryLocation (~/.kube/config) with no merging.
  1.  Determine the context to use based on the first hit in this chain
      1.  command line argument - the value of the `context` command line option
      1.  current-context from the merged kubeconfig file
      1.  Empty is allowed at this stage
  1.  Determine the cluster info and user to use.  At this point, we may or may not have a context.  They are built based on the first hit in this chain.  (run it twice, once for user, once for cluster)
      1.  command line argument - `user` for user name and `cluster` for cluster name
      1.  If context is present, then use the context's value
      1.  Empty is allowed
  1.  Determine the actual cluster info to use.  At this point, we may or may not have a cluster info.  Build each piece of the cluster info based on the chain (first hit wins):
      1.  command line arguments - `server`, `api-version`, `certificate-authority`, and `insecure-skip-tls-verify`
      1.  If cluster info is present and a value for the attribute is present, use it.
      1.  If you don't have a server location, error.
  1.  Determine the actual user info to use. User is built using the same rules as cluster info, EXCEPT that you can only have one authentication technique per user.
      1. Load precedence is 1) command line flag, 2) user fields from .kubeconfig, 3) kubernetes_auth file fields (if user has a `auth-path` or the `--auth-path` was provided)
      1. The command line flags are: `auth-path`, `client-certificate`, `client-key`, `username`, `password`, and `token`.
      1. If there are two conflicting techniques, fail.
  1.  For any information still missing, use default values and potentially prompt for authentication information

## Manipulation of .kubeconfig via `kubectl config <subcommand>`
In order to more easily manipulate .kubeconfig files, there are a series of subcommands to `kubectl config` to help.
```
kubectl config set-credentials name --auth-path=path/to/authfile --client-certificate=path/to/cert --client-key=path/to/key --token=string
  Sets a user entry in .kubeconfig.  If the referenced name already exists, it will be overwritten.
kubectl config set-cluster name --server=server --skip-tls=bool --certificate-authority=path/to/ca --api-version=string
  Sets a cluster entry in .kubeconfig.  If the referenced name already exists, it will be overwritten.
kubectl config set-context name --user=string --cluster=string --namespace=string
  Sets a config entry in .kubeconfig.  If the referenced name already exists, it will be overwritten.
kubectl config use-context name
  Sets current-context to name
kubectl config set property-name property-value
  Sets arbitrary value in .kubeconfig
kubectl config unset property-name
  Unsets arbitrary value in .kubeconfig
kubectl config view --local=true --global=false --kubeconfig=specific/filename --merged
  Displays the merged (or not) result of the specified .kubeconfig file

--local, --global, and --kubeconfig are valid flags for all of these operations.
```

### Example
```
$kubectl config set-credentials myself --auth-path=path/to/my/existing/auth-file
$kubectl config set-cluster local-server --server=http://localhost:8080
$kubectl config set-context default-context --cluster=local-server --user=myself
$kubectl config use-context default-context
$kubectl config set contexts.default-context.namespace the-right-prefix
$kubectl config view
```
produces this output
```
clusters:
  local-server:
    server: http://localhost:8080
contexts:
  default-context:
    cluster: local-server
    namespace: the-right-prefix
    user: myself
current-context: default-context
preferences: {}
users:
  myself:
    auth-path: path/to/my/existing/auth-file

```
and a .kubeconfig file that looks like this
```
apiVersion: v1
clusters:
- cluster:
    server: http://localhost:8080
  name: local-server
contexts:
- context:
    cluster: local-server
    namespace: the-right-prefix
    user: myself
  name: default-context
current-context: default-context
kind: Config
preferences: {}
users:
- name: myself
  user:
    auth-path: path/to/my/existing/auth-file
```

#### Commands for the example file
```
$kubectl config set preferences.colors true
$kubectl config set-cluster cow-cluster --server=http://cow.org:8080 --api-version=v1beta1
$kubectl config set-cluster horse-cluster --server=https://horse.org:4443 --certificate-authority=path/to/my/cafile
$kubectl config set-cluster pig-cluster --server=https://pig.org:443 --insecure-skip-tls-verify=true
$kubectl config set-credentials black-user --auth-path=path/to/my/existing/.kubernetes_auth_file
$kubectl config set-credentials blue-user --token=blue-token
$kubectl config set-credentials green-user --client-certificate=path/to/my/client/cert --client-key=path/to/my/client/key
$kubectl config set-context queen-anne-context --cluster=pig-cluster --user=black-user --namespace=saw-ns
$kubectl config set-context federal-context --cluster=horse-cluster --user=green-user --namespace=chisel-ns
$kubectl config use-context federal-context
```
