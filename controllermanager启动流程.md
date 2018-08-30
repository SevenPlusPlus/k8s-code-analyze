## ControllerManager启动流程
### 构建ControllerManagerCommand并启动
* cmd/kube-controller-manager/controller-manager.go:

```
func main() {
	command := app.NewControllerManagerCommand()

	if err := command.Execute(); err != nil {
		fmt.Fprintf(os.Stderr, "%v\n", err)
		os.Exit(1)
	}
}
```