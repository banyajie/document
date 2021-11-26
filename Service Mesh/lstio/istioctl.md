###### Istiojctl 命令

```shell
istioctl 
Available Commands:
  admin          Manage control plane (istiod) configuration
  analyze        Analyze Istio configuration and print validation messages
  authz          (authz is experimental. Use `istioctl experimental authz`)
  bug-report     Cluster information and log capture support tool.
  completion     generate the autocompletion script for the specified shell
  dashboard      Access to Istio web UIs
  experimental   Experimental commands that may be modified or deprecated
  help           Help about any command
  install        Applies an Istio manifest, installing or reconfiguring Istio on a cluster.
  kube-inject    Inject Istio sidecar into Kubernetes pod resources
  manifest       Commands related to Istio manifests
  operator       Commands related to Istio operator controller.
  profile        Commands related to Istio configuration profiles
  proxy-config   Retrieve information about proxy configuration from Envoy [kube only]
  proxy-status   Retrieves the synchronization status of each Envoy in the mesh [kube only]
  tag            Command group used to interact with revision tags
  upgrade        Upgrade Istio control plane in-place
  validate       Validate Istio policy and rules files
  verify-install Verifies Istio Installation Status
  version        Prints out build version information

Flags:
      --context string          The name of the kubeconfig context to use
  -h, --help                    help for istioctl
  -i, --istioNamespace string   Istio system namespace (default "istio-system")
  -c, --kubeconfig string       Kubernetes configuration file
  -n, --namespace string        Config namespace
      --vklog Level             number for the log level verbosity. Like -v flag. ex: --vklog=9
      
```

```shell
分析检查当前istio环境

istiooctl analyze

Info [IST0102] (Namespace default) The namespace is not enabled for Istio injection. Run 'kubectl label namespace default istio-injection=enabled' to enable it, or 'kubectl label namespace default istio-injection=disabled' to explicitly mark it as not needing injection.

注入：
kubectl label namespace default istio-injection=enabled
kubectl label namespace default istio-injection=disabled

单独注入
istioctl kube-inject -f deployment.yaml | kubectl apply -f - -n dev


```

```shell
istio profile
istio profile
Available Commands:
  diff        Diffs two Istio configuration profiles
  dump        Dumps an Istio configuration profile
  list        Lists available Istio configuration profiles


```

```shell

```



