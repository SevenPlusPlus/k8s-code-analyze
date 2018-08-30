## ControllerManager启动流程
### 构建ControllerManagerCommand并启动
* cmd/kube-controller-manager/controller-manager.go:

```
func main() {
	command := app.NewControllerManagerCommand()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

ControllerManagerCommand::Run
* cmd/kube-controller-manager/app/controllermanager.go:

```
Run: func(cmd *cobra.Command, args []string) {
 c, err := s.Config(KnownControllers(), ControllersDisabledByDefault.List())
 Run(c.Complete(), wait.NeverStop)
}
```
获取默认启用的的Controller列表
KnownControllers->NewControllerInitializers

```


```