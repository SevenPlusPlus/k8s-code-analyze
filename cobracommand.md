# cobra

---
cobra是一个命令行框架, 它可以生成和解析命令行工程.
---  
## 安装
cobra的源工程地址在github.com\/spf13\/cobra\/cobra,
同其他Go开源项目一样,我们通过go get来拉取:
```
$ go get -v github.com/spf13/cobra/cobra  
```
安装完成之后, 通过下面语句import到工程:  
```
Go import "github.com/spf13/cobra"
```
### 使用Cobra库
#### 创建 rootCmd
```
var RootCmd = &cobra.Command{  
Use: "hugo",   
Short: "Hugo is a very fast static site generator", 
Long: `A Fast and Flexible Static Site Generator built with love by spf13 and friends in Go. Complete documentation is available at http://hugo.spf13.com`,
Run: func(cmd *cobra.Command, args []string) {
 // Do Stuff Here }, 
} 
```
#### 定义并init cobra的参数
```
func init() {
 cobra.OnInitialize(initConfig)
 RootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra.yaml)")
 RootCmd.PersistentFlags().StringVarP(&projectBase, "projectbase", "b", "", "base project directory eg. github.com/spf13/")
 RootCmd.PersistentFlags().StringP("author", "a", "YOUR NAME", "Author name for copyright attribution")
 RootCmd.PersistentFlags().StringVarP(&userLicense, "license", "l", "", "Name of license for the project (can provide `licensetext` in config)")
 RootCmd.PersistentFlags().Bool("viper", true, "Use Viper for configuration")
 viper.BindPFlag("author", RootCmd.PersistentFlags().Lookup("author"))
 viper.BindPFlag("projectbase", RootCmd.PersistentFlags().Lookup("projectbase"))
 viper.BindPFlag("useViper", RootCmd.PersistentFlags().Lookup("viper"))
 viper.SetDefault("author", "NAME HERE <EMAIL ADDRESS>")
 viper.SetDefault("license", "apache")
}

func initConfig() {

 // Don't forget to read config either from cfgFile or from home directory!
 if cfgFile != "" {
 // Use config file from the flag.
 viper.SetConfigFile(cfgFile)
 } else {
 // Find home directory.
 home, err := homedir.Dir()
 if err != nil {
 fmt.Println(err)
 os.Exit(1)
 }

 // Search config in home directory with name ".cobra" (without extension).
 viper.AddConfigPath(home)
 viper.SetConfigName(".cobra")
 }
 if err := viper.ReadInConfig(); err != nil {
 fmt.Println("Can't read config:", err)
 os.Exit(1)
 }
}

```
#### 创建main.go
在Cobra应用程序中，通常main.go文件非常空白。 它有一个目的，来初始化Cobra 
```
package main

import (

 "fmt"

 "os"

 "{pathToYourApp}/cmd"

)

func main() {

 if err := cmd.RootCmd.Execute(); err != nil {

 fmt.Println(err)

 os.Exit(1)

 }

}
```
#### 新增子命令
如果你想创建一个版本命令，你可以创建cmd / version.go和 用下面的代码填充它：
```
package cmd

import (

 "github.com/spf13/cobra"

 "fmt"

)

func init() {

 RootCmd.AddCommand(versionCmd)

}

var versionCmd = &cobra.Command{

 Use: "version",

 Short: "Print the version number of Hugo",

 Long: `All software has versions. This is Hugo's`,

 Run: func(cmd *cobra.Command, args []string) {

 fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")

 },

}

```
#### 参数Flags工作  
##### 参数提供修饰符来控制动作命令的操作
##### 全局参数(Persistent Flags)
一个参数可以是“持久的”，这意味着这个参数将被分配的命令以及该命令下的每个命令都可用。 对于全局参数，在根上分配一个参数作为持久参数。
```
RootCmd.PersistentFlags().BoolVarP(&Verbose, "verbose", "v", false, "verbose output") 
```
##### 局部参数(Local Flags)
一个参数也可以在本地分配，只适用于该特定的命令。
```
RootCmd.Flags().StringVarP(&Source, "source", "s", "", "Source directory to read from") 
```
##### 绑定配置文件中的参数
你可以通过viper来绑定你的参数:
```
var author string 
func init() {
  RootCmd.PersistentFlags().StringVar(&author, "author", "YOUR NAME", "Author name for copyright attribution") 
  viper.BindPFlag("author", RootCmd.PersistentFlags().Lookup("author")) 
}
``` 
##### 参数的校验
有如下校验器：
* NoArgs - the command will report an error if there are any positional args.
* ArbitraryArgs - the command will accept any args.
* OnlyValidArgs - the command will report an error if there are any positional args that are not in the ValidArgs field of Command.
* MinimumNArgs(int) - the command will report an error if there are not at least N positional args.
* MaximumNArgs(int) - the command will report an error if there are more than N positional args.
* ExactArgs(int) - the command will report an error if there are not exactly N positional args.
* RangeArgs(min, max) - the command will report an error if the number of args is not between the minimum and maximum number of expected args.

#### 下面是一个完整的cobra例子
```
package main

import (

 "fmt"

 "strings"

 "github.com/spf13/cobra"

)

func main() {

 var echoTimes int

 var cmdPrint = &cobra.Command{

 Use: "print [string to print]",

 Short: "Print anything to the screen",

 Long: `print is for printing anything back to the screen.

For many years people have printed back to the screen.`,

 Args: cobra.MinimumNArgs(1),

 Run: func(cmd *cobra.Command, args []string) {

 fmt.Println("Print: " + strings.Join(args, " "))

 },

 }

 var cmdEcho = &cobra.Command{

 Use: "echo [string to echo]",

 Short: "Echo anything to the screen",

 Long: `echo is for echoing anything back.

Echo works a lot like print, except it has a child command.`,

 Args: cobra.MinimumNArgs(1),

 Run: func(cmd *cobra.Command, args []string) {

 fmt.Println("Print: " + strings.Join(args, " "))

 },

 }

 var cmdTimes = &cobra.Command{

 Use: "times [# times] [string to echo]",

 Short: "Echo anything to the screen more times",

 Long: `echo things multiple times back to the user by providing

a count and a string.`,

 Args: cobra.MinimumNArgs(1),

 Run: func(cmd *cobra.Command, args []string) {

 for i := 0; i < echoTimes; i++ {

 fmt.Println("Echo: " + strings.Join(args, " "))

 }

 },

 }

 cmdTimes.Flags().IntVarP(&echoTimes, "times", "t", 1, "times to echo the input")

 var rootCmd = &cobra.Command{Use: "app"}

 rootCmd.AddCommand(cmdPrint, cmdEcho)

 cmdEcho.AddCommand(cmdTimes)

 rootCmd.Execute()

}
```
#### PreRun and PostRun 钩子

可以在命令的主**_Run_**功能之前或之后运行功能。 PersistentPreRun和PreRun函数将在Run之前执行。 PersistentPostRun和PostRun将在Run之后执行。 如果Persistent \* Run函数没有声明自己，那么Persistent \* Run函数将被子代继承。 这些功能按以下顺序运行：

1. PersistentPreRun
2. PreRun
3. Run
4. PostRun
5. PersistentPostRun


