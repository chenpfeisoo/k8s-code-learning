## dynamic client 
dynamic client也是对rest client的封装，不同之处是dynamic client 不仅能访问k8s内置资源还可以访问crd资源

### 结构体定义
```go
type dynamicClient struct {
	client *rest.RESTClient
}
```
```go
type dynamicResourceClient struct {
	client    *dynamicClient
	namespace string
	resource  schema.GroupVersionResource
}
```
### 接口定义
```go
type ResourceInterface interface {
	Create(ctx context.Context, obj *unstructured.Unstructured, options metav1.CreateOptions, subresources ...string) (*unstructured.Unstructured, error)
	Update(ctx context.Context, obj *unstructured.Unstructured, options metav1.UpdateOptions, subresources ...string) (*unstructured.Unstructured, error)
	UpdateStatus(ctx context.Context, obj *unstructured.Unstructured, options metav1.UpdateOptions) (*unstructured.Unstructured, error)
	Delete(ctx context.Context, name string, options metav1.DeleteOptions, subresources ...string) error
	DeleteCollection(ctx context.Context, options metav1.DeleteOptions, listOptions metav1.ListOptions) error
	Get(ctx context.Context, name string, options metav1.GetOptions, subresources ...string) (*unstructured.Unstructured, error)
	List(ctx context.Context, opts metav1.ListOptions) (*unstructured.UnstructuredList, error)
	Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
	Patch(ctx context.Context, name string, pt types.PatchType, data []byte, options metav1.PatchOptions, subresources ...string) (*unstructured.Unstructured, error)
}
```
dynamicResourceClient  实现了以上的ResourceInterface 接口方法，可以对GroupVersionResource 注册的资源进行接口方法操作
在对资源类型进行注册的时候有以下集中注册表
```go
var watchScheme = runtime.NewScheme()
var basicScheme = runtime.NewScheme()
var deleteScheme = runtime.NewScheme()
var parameterScheme = runtime.NewScheme()
```
我们看到 dynamic client 实现了informer 和 lisener
```go
type dynamicInformer struct {
	informer cache.SharedIndexInformer
	gvr      schema.GroupVersionResource
}
```
informer star之后发现资源变化时要落缓存
```go
func (f *dynamicSharedInformerFactory) WaitForCacheSync(stopCh <-chan struct{}) map[schema.GroupVersionResource]bool {
	informers := func() map[schema.GroupVersionResource]cache.SharedIndexInformer {
		f.lock.Lock()
		defer f.lock.Unlock()

		informers := map[schema.GroupVersionResource]cache.SharedIndexInformer{}
		for informerType, informer := range f.informers {
			if f.startedInformers[informerType] {
				informers[informerType] = informer.Informer()
			}
		}
		return informers
	}()

	res := map[schema.GroupVersionResource]bool{}
	for informType, informer := range informers {
		res[informType] = cache.WaitForCacheSync(stopCh, informer.HasSynced)
	}
	return res
}
```