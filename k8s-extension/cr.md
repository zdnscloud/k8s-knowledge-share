# Custom Resource
## Resource Handler
1. aggregated API server
2. apiextensions-apiserver

## Custom Resource Definition(CRD)
1. apiVersion, kindd, metadata, spec, status
2. validation support in OpenAPI v3 schema
3. short names and categories
4. printer columes, kubectl use server-side printing to render the output
```yaml
additionalPrinterColumns: 
- name: schedule
  type: string
  JSONPath: .spec.schedule
- name: command
  type: string
  JSONPath: .spec.command
- name: phase
  type: string
  JSONPath: .status.phase
```
5. subresources

## CRD with operator-sdk
1. generate the project scaffold
```bash
operator-sdk new immense --vendor --dep-manager dep
```
2. generate go structure and interface implementation
```bash
operator-sdk add api --api-version=zdns.cn/v1 --kind=Cluster
```
3. update spec and status of the resource 
```bash
operator-sdk generate k8s
operator-sdk generate openapi
```
4. Generate clientset and informer
```bash
go get k8s.io/code-generator
go get k8s.io/apimachinery
k8s.io/code-generate/generate-groups.sh all \
        "vg-op/pkg/client" \
        "vg-op/pkg/apis" \
        zdns:v1
```

## Dynamic client 
1. client without scheme or REST mapping
```golang
gvr := schema.GroupVersionResource{Group: "apps", Version: "v1", Resource: "deployments"}
client, err := NewForConfig(cfg)
u, err := client.Resource(gvr).Namespace(namespace).Get("foo", metav1.GetOptions{})
name, found, err := unstructurd.NestedString(u.Object, "metadata", "name")
```
2. object format
  * objects are represented by `map[stirng]interface{}`
  * arrays by `[]interface{}`
  * primitive types are `string`, `bool`, `float64` and `int64`
