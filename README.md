# Config

Golang's application config manage tool library. 

> **WEMPY CONFIG HELPERS**

## Features

- Support multi format: `JSON`(default), `INI`, `YAML`, `TOML`, `HCL`, `ENV`, `Flags`
  - `JSON` content support comments. will auto clear comments
  - Other drivers are used on demand, not used will not be loaded into the application.
	- Possibility to add custom driver for your specific format
- Support multi-file and multi-data loading
- Support loading configuration from os ENV
- Support for loading configuration data from remote URLs
- Support for setting configuration data from command line arguments(`flags`)
- Support data overlay and merge, automatically load by key when loading multiple copies of data
- Support for binding all or part of the configuration data to the structure
- Support get sub value by path, like `map.key` `arr.2`
- Support parse ENV name and allow with default value. like `envKey: ${SHELL|/bin/bash}` -> `envKey: /bin/zsh`
- Generic api `Get` `Int` `Uint` `Int64` `Float` `String` `Bool` `Ints` `IntMap` `Strings` `StringMap` ...
- Complete unit test(code coverage > 95%)

> Provide a sub-package `dotenv` that supports importing data from files (eg `.env`) to ENV

## Usage

Here using the yaml format as an example(`testdata/yml_other.yml`):

```yaml
name: app2
debug: false
baseKey: value2
shell: ${SHELL}
envKey1: ${NotExist|defValue}

map1:
    key: val2
    key2: val20

arr1:
    - val1
    - val21
```

### Load data

> examples code please see [_examples/yaml.go](_examples/yaml.go):

```go
package main

import (
    "github.com/wempy-id/utility/config/v2"
    "github.com/wempy-id/utility/config/v2/yaml"
)

// go run ./examples/yaml.go
func main() {
	config.WithOptions(config.ParseEnv)
	
	// add driver for support yaml content
	config.AddDriver(yaml.Driver)
	// config.SetDecoder(config.Yaml, yaml.Decoder)

	err := config.LoadFiles("testdata/yml_base.yml")
	if err != nil {
		panic(err)
	}

	// load more files
	err = config.LoadFiles("testdata/yml_other.yml")
	// can also load multi at once
	// err := config.LoadFiles("testdata/yml_base.yml", "testdata/yml_other.yml")
	if err != nil {
		panic(err)
	}
	
	// fmt.Printf("config data: \n %#v\n", config.Data())
}
```

## Map Data To Structure

> Note: The default binding mapping tag of a structure is `mapstructure`, which can be changed by setting `Options.TagName`

```go
user := struct {
    Age  int
    Kye  string
    UserName  string `mapstructure:"user_name"`
    Tags []int
}{}
err = config.BindStruct("user", &user)

fmt.Println(user.UserName) // inhere
```

### Direct Read data

- Get integer

```go
age := config.Int("age")
fmt.Print(age) // 100
```

- Get bool

```go
val := config.Bool("debug")
fmt.Print(val) // true
```

- Get string

```go
name := config.String("name")
fmt.Print(name) // inhere
```

- Get strings(slice)

```go
arr1 := config.Strings("arr1")
fmt.Printf("%#v", arr1) // []string{"val1", "val21"}
```

- Get string map

```go
val := config.StringMap("map1")
fmt.Printf("%#v",val) // map[string]string{"key":"val2", "key2":"val20"}
```

- Value contains ENV var

```go
value := config.String("shell")
fmt.Print(value) // "/bin/zsh"
```

- Get value by key path

```go
// from array
value := config.String("arr1.0")
fmt.Print(value) // "val1"

// from map
value := config.String("map1.key")
fmt.Print(value) // "val2"
```

- Setting new value

```go
// set value
config.Set("name", "new name")
name = config.String("name")
fmt.Print(name) // "new name"
```

## Load from flags

> Support simple flags parameter parsing, loading

```go
// flags like: --name inhere --env dev --age 99 --debug

// load flag info
keys := []string{"name", "env", "age:int" "debug:bool"}
err := config.LoadFlags(keys)

// read
config.String("name") // "inhere"
config.String("env") // "dev"
config.Int("age") // 99
config.Bool("debug") // true
```

## Load from ENV

```go
// os env: APP_NAME=config APP_DEBUG=true
// load ENV info
config.LoadOSEnv([]string{"APP_NAME", "APP_NAME"}, true)

// read
config.Bool("app_debug") // true
config.String("app_name") // "config"
```

## New Config Instance

You can create custom config instance

```go
// create new instance, will auto register JSON driver
myConf := config.New("my-conf")

// create empty instance
myConf := config.NewEmpty("my-conf")

// create and with some options
myConf := config.NewWithOptions("my-conf", config.ParseEnv, config.ReadOnly)
```

## Available Options

```go
// Options config options
type Options struct {
	// parse env value. like: "${EnvName}" "${EnvName|default}"
	ParseEnv bool
	// config is readonly. default is False
	Readonly bool
	// enable config data cache. default is False
	EnableCache bool
	// parse key, allow find value by key path. default is True eg: 'key.sub' will find `map[key]sub`
	ParseKey bool
	// tag name for binding data to struct
	TagName string
	// the delimiter char for split key, when `FindByPath=true`. default is '.'
	Delimiter byte
	// default write format. default is JSON
	DumpFormat string
	// default input format. default is JSON
	ReadFormat string
}
```

## API Methods Refer

### Load Config

- `LoadOSEnv(keys []string)` Load from os ENV
- `LoadData(dataSource ...interface{}) (err error)` Load from struts or maps
- `LoadFlags(keys []string) (err error)` Load from CLI flags
- `LoadExists(sourceFiles ...string) (err error)` 
- `LoadFiles(sourceFiles ...string) (err error)`
- `LoadRemote(format, url string) (err error)`
- `LoadSources(format string, src []byte, more ...[]byte) (err error)`
- `LoadStrings(format string, str string, more ...string) (err error)`

### Getting Values

- `Bool(key string, defVal ...bool) bool`
- `Int(key string, defVal ...int) int`
- `Uint(key string, defVal ...uint) uint`
- `Int64(key string, defVal ...int64) int64`
- `Ints(key string) (arr []int)`
- `IntMap(key string) (mp map[string]int)`
- `Float(key string, defVal ...float64) float64`
- `String(key string, defVal ...string) string`
- `Strings(key string) (arr []string)`
- `StringMap(key string) (mp map[string]string)`
- `Get(key string, findByPath ...bool) (value interface{})`

### Setting Values

- `Set(key string, val interface{}, setByPath ...bool) (err error)`

### Useful Methods

- `Getenv(name string, defVal ...string) (val string)`
- `AddDriver(driver Driver)`
- `Data() map[string]interface{}`
- `SetData(data map[string]interface{})` set data to override the Config.Data
- `Exists(key string, findByPath ...bool) bool`
- `DumpTo(out io.Writer, format string) (n int64, err error)`
- `BindStruct(key string, dst interface{}) error`

## Run Tests

```bash
go test -cover
// contains all sub-folder
go test -cover ./...
```

## License

**MIT**
