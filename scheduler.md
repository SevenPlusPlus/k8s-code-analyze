## scheduler启动流程
### 服务启动流程
* cmd/kube-scheduler/scheduler.go:
```
func main() {
   command := app.NewSchedulerCommand()
   command.Execute()
}
```
新建SchedulerCommand
* cmd/kube-scheduler/app/server.go:

