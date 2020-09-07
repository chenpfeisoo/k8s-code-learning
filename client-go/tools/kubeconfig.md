## kubeconfig

### 从yaml看起
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1
    server: https://test.com
  name: cls-pzj2ebcs
contexts:
- context:
    cluster: cls-pzj2ebcs
    user: "100002743387"
  name: test-context-default
current-context: test-default
kind: Config
preferences: {}
users:
- name: "100002743387"
  user:
    client-certificate-data: LS0tLS1CRUdJTi
    client-key-data: LS0tLS1CRUdJT
```
### kubeconfig 对象定义
```go
type Config struct {
	// Legacy field from pkg/api/types.go TypeMeta.
	// TODO(jlowdermilk): remove this after eliminating downstream dependencies.
	// +k8s:conversion-gen=false
	// +optional
	Kind string `json:"kind,omitempty"`
	// Legacy field from pkg/api/types.go TypeMeta.
	// TODO(jlowdermilk): remove this after eliminating downstream dependencies.
	// +k8s:conversion-gen=false
	// +optional
	APIVersion string `json:"apiVersion,omitempty"`
	// Preferences holds general information to be use for cli interactions
	Preferences Preferences `json:"preferences"`
	// Clusters is a map of referencable names to cluster configs
	Clusters map[string]*Cluster `json:"clusters"`
	// AuthInfos is a map of referencable names to user configs
	AuthInfos map[string]*AuthInfo `json:"users"`
	// Contexts is a map of referencable names to context configs
	Contexts map[string]*Context `json:"contexts"`
	// CurrentContext is the name of the context that you would like to use by default
	CurrentContext string `json:"current-context"`
	// Extensions holds additional information. This is useful for extenders so that reads and writes don't clobber unknown fields
	// +optional
	Extensions map[string]runtime.Object `json:"extensions,omitempty"`
}
```
### 生成一个kubeconfig
```go
func getConfigFromFile(filename string) (*clientcmdapi.Config, error) {
	config, err := LoadFromFile(filename)
	if err != nil && !os.IsNotExist(err) {
		return nil, err
	}
	if config == nil {
		config = clientcmdapi.NewConfig()
	}
	return config, nil
}
```
默认的路径
kubeconfig 默认在home/.kube/kubeconfig
```go
const (
	RecommendedConfigPathFlag   = "kubeconfig"
	RecommendedConfigPathEnvVar = "KUBECONFIG"
	RecommendedHomeDir          = ".kube"
	RecommendedFileName         = "config"
	RecommendedSchemaName       = "schema"
)

var (
	RecommendedConfigDir  = path.Join(homedir.HomeDir(), RecommendedHomeDir)
	RecommendedHomeFile   = path.Join(RecommendedConfigDir, RecommendedFileName)
	RecommendedSchemaFile = path.Join(RecommendedConfigDir, RecommendedSchemaName)
)

```
kubeconfig有一个很重要的功能就是 多配置信息merge生成一份多配置信息列表
整个过程是由func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error)  方法控制

```go
func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error) {
    	if len(rules.ExplicitPath) > 0 {
		if _, err := os.Stat(rules.ExplicitPath); os.IsNotExist(err) {
			return nil, err
		}
		kubeConfigFiles = append(kubeConfigFiles, rules.ExplicitPath)

	} else {
		kubeConfigFiles = append(kubeConfigFiles, rules.Precedence...)
	}

    kubeconfigs := []*clientcmdapi.Config{}
    ......
    	// first merge all of our maps
	mapConfig := clientcmdapi.NewConfig()

	for _, kubeconfig := range kubeconfigs {
		mergo.MergeWithOverwrite(mapConfig, kubeconfig)
	}

	// merge all of the struct values in the reverse order so that priority is given correctly
	// errors are not added to the list the second time
	nonMapConfig := clientcmdapi.NewConfig()
	for i := len(kubeconfigs) - 1; i >= 0; i-- {
		kubeconfig := kubeconfigs[i]
		mergo.MergeWithOverwrite(nonMapConfig, kubeconfig)
	}

	// since values are overwritten, but maps values are not, we can merge the non-map config on top of the map config and
	// get the values we expect.
	config := clientcmdapi.NewConfig()
	mergo.MergeWithOverwrite(config, mapConfig)
	mergo.MergeWithOverwrite(config, nonMapConfig)

	if rules.ResolvePaths() {
		if err := ResolveLocalPaths(config); err != nil {
			errlist = append(errlist, err)
		}
	}
	return config, utilerrors.NewAggregate(errlist)
}
```