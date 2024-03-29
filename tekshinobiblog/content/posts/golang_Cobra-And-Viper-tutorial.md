---
title: "Golang:Cobra and Viper Tutorial"
date: 2021-09-23T13:04:41+03:00
draft: false 
categories: ["golang"]
tags: ["tools"]
---

Cobra is a tool for making CLI application. Viper is a configuration management solution for Go applications which allows you to specify configuration options for your application in several ways, including **configuration files, environment variables, and command-line flags**.

## Why viper for config management?

- It can find, load, and unmarshal values from a config file.
- It supports many types of files, such as JSON, TOML, YAML, ENV, or INI.
- It can also read values from environment variables or command-line flags.
- It gives us the ability to set or override default values.
- Moreover, if you prefer to store your settings in a remote system such as Etcd or Consul, then you can use viper to read data from them directly.
- It works for both unencrypted and encrypted values.
- One more interesting thing about Viper is, it can watch for changes in the config file, and notify the application about it.
- We can also use viper to save any config modification we made to the file.

## Let's do it

Install cobra: `go install github.com/spf13/cobra`
Assuming that this is a blank project with just `go.mod` file, create the boilerplate by:
```shell
cobra-cli init
```
Your project structure will now look like this:
```shell
.
├── cmd
│   └── root.go
├── go.mod
├── go.sum
├── LICENSE
└──  main.go
```
Next, let’s update the cmd/root.go file. This file is where your base command and global flags are set up.

In `cmd/root.go`, you will see something like this
```go
var rootCmd = &cobra.Command{
	Use:   "viper_cobra_demo",
	Short: "root command",
	Long:  `root command.`,
}
```
this is the root command. This command automatically runs when you call `go run main.go`. Since root command is also a command, you need to provide the `Use` parameter, but you don't use it as this gets invoked automatically. All other commands are added to this root command as sub-commands.

Let's add our first sub-command.
`cobra-cli add server`

This will add a subcommand called server and also generate boilerplate (inside `cmd/server.go`) along with adding it to the root-command.

Now you can invoke this subcommand like so:
`go run main.go server`

Now, we will also start organizing our configuration.

create a folder `config` in you project root and create a file in it `config.go` with these contents:
`config/config.go`:
```go
package config

type Config struct {
	PublicHTTPPort string
}
```

Here our configuration is stored in a struct called Config. Currently, for demo, it has one field (PublicHTTPPort). In real app, it will have many more fields like database configuration, cache configuration etc.

Coming back to `cmd/server.go`
Here are some more additions:
```go
// serverCmd represents the server command
var serverCmd = &cobra.Command{
	Use:   "server",
	Short: "Start demo server",
	Long:  `Start demo server.`,
	PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
		return initializeConfig(cmd)
	},
	Run: func(cmd *cobra.Command, args []string) {
		fmt.Println("server called")
	},
}

func init() {
	rootCmd.AddCommand(serverCmd)
	serverCmd.PersistentFlags().StringVar(&Cfg.PublicHTTPPort, "public-http-port", "8081", "public HTTP port")
}

func initializeConfig(cmd *cobra.Command) error {
	v := viper.New()
	v.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))
	v.AutomaticEnv()
	cmd.Flags().VisitAll(func(f *pflag.Flag) {
		configName := f.Name
		if !f.Changed && v.IsSet(configName) {
			val := v.Get(configName)
			_ = cmd.Flags().Set(f.Name, fmt.Sprintf("%v", val))
		}
	})

    /* only for demo*/
	fmt.Printf("the port:%s\n", v.Get("public-http-port"))
    /* only for demo*/
	fmt.Printf("the port from cfg:%s\n", Cfg.PublicHTTPPort)
	
    return nil
}
```

The important part is the function called in the pre-run hook:`initializeConfig`:
Some things to note. Viper allows us to manage configurations supplied via command-line args, environment variables and config files. A common paradigm is that of the 3, only two are supported.. like config-file + env-vars OR command-line-args + env-vars. It's so because its not a good pattern to spread configuration in too many places. Here, we are supporting command-line-args + env-vars.

Reading values from file/command-line-args allows us to easily specify default configuration for local development and testing.

While reading values from environment variables will help us override the default settings when deploying our application to staging or production using docker containers.

## command-line-args + env-vars approach

`v := viper.New()` creates a new viper object.

`v.SetEnvKeyReplacer(strings.NewReplacer("-", "_"))` : this line is more tricky to understand. So, here's the explanation:
Let's say you launched this code like so:
```shell
PUBLIC_HTTP_PORT=9091 go run main.go  server
```
Here we are setting env-var `PUBLIC_HTTP_PORT` when running the app.
Now look at this line:`fmt.Printf("the port:%s\n", v.Get("public-http-port"))`
Here, we are trying to fetch the configuration variable with the key `public-http-port`. Now the `SetEnvKeyReplacer` line appears to make sense. When you use `SetEnvKeyReplacer`, you declare to replace `_` with `-` whenever a `v.Get` operation happens. Hence, viper internally takes care to convert uppercase to lower case and replace `_` with `-` before comparing `PUBLIC_HTTP_PORT` with the supplied key in the `v.Get("public-http-port")`. Hence, `SetEnvKeyReplacer` is related to `v.Get` action. 

Some details about `v.Get`: Since, `v` is a viper object, `v.Get("key")` is a way to fetch value of environment variable named `key`. 

Next line is: `v.AutomaticEnv()`: This line means, read the currently set environment variables and set any that match (like `public-http-port` in the previous example). **Note that this way, you will be able to get the env-var via `v.Get` but it will not get stored in the corresponding `Cfg` struct field.**. This makes sense since at this point, environment variables are loaded into viper object but since Cfg is where Cobra comand flags store values, nothing gets stored there as `v.AutomaticEnv()` only loads the viper object.

Next block of code related to `cmd.Flags().VisitAll`: This is where we synchronize viper object's values with those stored in `Cfg`. It will soon make sense. Read on...: It iterates over each cobra flag (here there are 2: http-public-port, help) and if they don't have a value set, but a corresponding setting exists in viper (like it was supplied like soe `PUBLIC_HTTP_SERVER=8021 go run main.go server`), copy that into the flag `public-http-server` and also store that value to the corresponding field in `Cfg`. **Note that this is how cobra+viper overrides a default provided flag value** (8081 in this case) **with that one loaded from the environment** (8021 in this case). So, in this case `Cfg.PublicHTTPPort` value will be `8021`.
Here, for `f.Name` = `public-http-port`, `v.IsSet(configName)` will be true (configname is `public-http-port`) and since we have set the `PUBLIC_HTTP_SERVER=8021`, `v.IsSet` will be true. `f.Changed` will be false as we did not set this flag from command line (its set to default value), so `!f.Changed` will be true. And then this line `_ = cmd.Flags().Set(f.Name, fmt.Sprintf("%v", val))` will cause the corresponding flag (`public-http-port`) set to 8021, which means `Cfg.PublicHTTPPort` is set to `8021` as that is where the flag's value is set (remember.. this flag was created as StringVar type, with its value persisted in `Cfg.PublicHTTPPort`).

lastly, note this line from `init()`:
`serverCmd.PersistentFlags().StringVar(&Cfg.PublicHTTPPort, "public-http-port", "8081", "public HTTP port")`: here we are storing the value passed to command line arguement `public-http-port` into its corresponding field in `Cfg`. In other words, `Cfg` is a struct where we store all values passed via command line arguements to access them later. The `Cfg` struct is a mechanism through which we can access values passed to cobra commands via command line arguements.

## config-file + env-vars approach

first create a struct to store the config file path that will be supplied via command line args. So, in `config/Config.go`:
```go
package config

type Config struct {
	PublicHTTPPort string `mapstructure:"PUBLIC_HTTP_PORT"`
}

type CfgFile struct {
    Path string
    Name string
    Type string
}
``` 
Notice that `Config` struct has `mapstructure` tag added. This is because in order to get the value of the variables from config file and store them in this struct, we need to use the unmarshaling feature of Viper. Viper uses the `mapstructure` package under the hood for unmarshaling values, so we use the mapstructure tags to specify the name of each config field. Also, we must use the exact name of each variable as being declared in the supplied config file. So, if in the file its `PUBLIC_HTTP_PORT`, then use exact same value as tag name for `PublicHTTPPort`

declare `var CfgF config.CfgFile` globally, at top of `cmd/server.go` file

Rest is nearly the same as we saw above. The only difference being in `init()` and `initializeConfig()` 

`init()`:
```go
func init() {
	rootCmd.AddCommand(serverCmd)
	serverCmd.PersistentFlags().StringVar(&CfgF.Path, "config-file-path", ".", "confuguration file path")
	serverCmd.PersistentFlags().StringVar(&CfgF.Name, "config-file-name", "cfg", "confuguration file name")
	serverCmd.PersistentFlags().StringVar(&CfgF.Type, "config-file-type", "env", "confuguration file type")
}
```

and here is our `initializeConfig`:
```go
func initializeConfig(cmd *cobra.Command) error {
	v := viper.New()
	v.AddConfigPath(CfgFile.Path)
	v.SetConfigName(CfgFile.Name)
	v.SetConfigType(CfgFile.Type)

	v.AutomaticEnv()

	if err := v.ReadInConfig(); err != nil {
		return err
	}

	if err := v.Unmarshal(&Cfg); err != nil {
		return err
	}
    /* only for demo */
	fmt.Printf("the port:%s\n", v.Get("PUBLIC_HTTP_PORT"))
    /* only for demo */
	fmt.Printf("the port in Cfg:%#v\n", Cfg)
	return nil
}
```

First, we call v.AddConfigPath() to tell Viper the location of the config file. In this case, the location is given by the input path argument.

Next, we call v.SetConfigName() to tell Viper to look for a config file with a specific name. Our config file is app.env, so its name is app.

We also tell Viper the type of the config file, which is env in this case, by calling v.SetConfigFile() and pass in env. You can also use JSON, XML or any other format here if you want, just make sure your config file has the correct format and extension.

Now, besides reading configurations from file, we also want viper to read values from environment variables. So we call v.AutomaticEnv() to tell viper to automatically override values that it has read from config file with the values of the corresponding environment variables if they exist.

After that, we call v.ReadInConfig() to start reading config values. If error is not nil, then we simply return it.

Otherwise, we call v.Unmarshal() to unmarshal the values into the target config object. And finally just return the config object and any error if it occurs.

## Using the configuration values

Now, that we have configuration values loaded into our target configuration object, `Cfg`, it will be available to use in the function passed to `Run`
```go
Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("server called")
},
```

You can write your code in this method that needs these config values.

Note that you can access the values read into viper from environment directly via `v.Get`, doing so is not considered a good practise. Rather inject them from this method (one with `Run`). This is good for testability.