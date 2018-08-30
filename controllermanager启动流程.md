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
func KnownControllers() []string { ret := sets.StringKeySet(NewControllerInitializers(IncludeCloudLoops)) // add "special" controllers that aren't initialized normally. These controllers cannot be initialized // using a normal function. The only known special case is the SA token controller which *must* be started // first to ensure that the SA tokens for future controllers will exist. Think very carefully before adding // to this list. ret.Insert( saTokenControllerName, ) return ret.List()}

```