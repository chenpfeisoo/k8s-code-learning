##   第二节clientset  
 https://github.com/chenpfeisoo/client-go/tree/master/kubernetes 
 中提供了k8s clientset客户端实现，clientset主要访问k8s内置资源
 clientset 包含了k8s所有的api group,我们看clientset的结构体定义
```go
type Clientset struct {
	*discovery.DiscoveryClient
	admissionregistrationV1      *admissionregistrationv1.AdmissionregistrationV1Client
	admissionregistrationV1beta1 *admissionregistrationv1beta1.AdmissionregistrationV1beta1Client
	appsV1                       *appsv1.AppsV1Client
	appsV1beta1                  *appsv1beta1.AppsV1beta1Client
	appsV1beta2                  *appsv1beta2.AppsV1beta2Client
	authenticationV1             *authenticationv1.AuthenticationV1Client
	authenticationV1beta1        *authenticationv1beta1.AuthenticationV1beta1Client
	authorizationV1              *authorizationv1.AuthorizationV1Client
	authorizationV1beta1         *authorizationv1beta1.AuthorizationV1beta1Client
	autoscalingV1                *autoscalingv1.AutoscalingV1Client
	autoscalingV2beta1           *autoscalingv2beta1.AutoscalingV2beta1Client
	autoscalingV2beta2           *autoscalingv2beta2.AutoscalingV2beta2Client
	batchV1                      *batchv1.BatchV1Client
	batchV1beta1                 *batchv1beta1.BatchV1beta1Client
	batchV2alpha1                *batchv2alpha1.BatchV2alpha1Client
	certificatesV1               *certificatesv1.CertificatesV1Client
	certificatesV1beta1          *certificatesv1beta1.CertificatesV1beta1Client
	coordinationV1beta1          *coordinationv1beta1.CoordinationV1beta1Client
	coordinationV1               *coordinationv1.CoordinationV1Client
	coreV1                       *corev1.CoreV1Client
	discoveryV1alpha1            *discoveryv1alpha1.DiscoveryV1alpha1Client
	discoveryV1beta1             *discoveryv1beta1.DiscoveryV1beta1Client
	eventsV1                     *eventsv1.EventsV1Client
	eventsV1beta1                *eventsv1beta1.EventsV1beta1Client
	extensionsV1beta1            *extensionsv1beta1.ExtensionsV1beta1Client
	flowcontrolV1alpha1          *flowcontrolv1alpha1.FlowcontrolV1alpha1Client
	networkingV1                 *networkingv1.NetworkingV1Client
	networkingV1beta1            *networkingv1beta1.NetworkingV1beta1Client
	nodeV1alpha1                 *nodev1alpha1.NodeV1alpha1Client
	nodeV1beta1                  *nodev1beta1.NodeV1beta1Client
	policyV1beta1                *policyv1beta1.PolicyV1beta1Client
	rbacV1                       *rbacv1.RbacV1Client
	rbacV1beta1                  *rbacv1beta1.RbacV1beta1Client
	rbacV1alpha1                 *rbacv1alpha1.RbacV1alpha1Client
	schedulingV1alpha1           *schedulingv1alpha1.SchedulingV1alpha1Client
	schedulingV1beta1            *schedulingv1beta1.SchedulingV1beta1Client
	schedulingV1                 *schedulingv1.SchedulingV1Client
	settingsV1alpha1             *settingsv1alpha1.SettingsV1alpha1Client
	storageV1beta1               *storagev1beta1.StorageV1beta1Client
	storageV1                    *storagev1.StorageV1Client
	storageV1alpha1              *storagev1alpha1.StorageV1alpha1Client
}
```
clientset 对象的生成有两种方式 
1.通过传入的config对象生成，与restclient方式一致
```go
func NewForConfig(c *rest.Config) (*Clientset, error) {
	configShallowCopy := *c
	if configShallowCopy.RateLimiter == nil && configShallowCopy.QPS > 0 {
		if configShallowCopy.Burst <= 0 {
			return nil, fmt.Errorf("burst is required to be greater than 0 when RateLimiter is not set and QPS is set to greater than 0")
		}
		configShallowCopy.RateLimiter = flowcontrol.NewTokenBucketRateLimiter(configShallowCopy.QPS, configShallowCopy.Burst)
	}
	var cs Clientset
	var err error
	cs.admissionregistrationV1, err = admissionregistrationv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.admissionregistrationV1beta1, err = admissionregistrationv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.appsV1, err = appsv1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	cs.appsV1beta1, err = appsv1beta1.NewForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	......
	cs.DiscoveryClient, err = discovery.NewDiscoveryClientForConfig(&configShallowCopy)
	if err != nil {
		return nil, err
	}
	return &cs, nil
}
```
2.通过传入的restclient对象生成
```go
func New(c rest.Interface) *Clientset {
	var cs Clientset
	cs.admissionregistrationV1 = admissionregistrationv1.New(c)
	cs.admissionregistrationV1beta1 = admissionregistrationv1beta1.New(c)
	cs.appsV1 = appsv1.New(c)
	cs.appsV1beta1 = appsv1beta1.New(c)
	cs.appsV1beta2 = appsv1beta2.New(c)
	cs.authenticationV1 = authenticationv1.New(c)
	cs.authenticationV1beta1 = authenticationv1beta1.New(c)
	cs.authorizationV1 = authorizationv1.New(c)
	cs.authorizationV1beta1 = authorizationv1beta1.New(c)
	cs.autoscalingV1 = autoscalingv1.New(c)
	cs.autoscalingV2beta1 = autoscalingv2beta1.New(c)
	cs.autoscalingV2beta2 = autoscalingv2beta2.New(c)
	cs.batchV1 = batchv1.New(c)
    ......
	cs.storageV1 = storagev1.New(c)
	cs.storageV1alpha1 = storagev1alpha1.New(c)

	cs.DiscoveryClient = discovery.NewDiscoveryClient(c)
	return &cs
}
```
##  资源注册
在clientset的代码路径里面我们还看到了一个代码路径
https://github.com/chenpfeisoo/client-go/blob/master/kubernetes/scheme/register.go
register主要是对k8s资源进行注册,整个注册是放在init里面完成的，关于Scheme注册单独开一个章节说明
```go
var AddToScheme = localSchemeBuilder.AddToScheme

func init() {
	v1.AddToGroupVersion(Scheme, schema.GroupVersion{Version: "v1"})
	utilruntime.Must(AddToScheme(Scheme))
}
```