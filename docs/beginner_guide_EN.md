Welcome to the chaosblade project, this article will take you through the use of the chaosblade tool

# download chaosblade

Get the latest [release](https://github.com/chaosblade-io/chaosblade/releases) package for chaosblade. Currently supported platforms are linux/amd64 and darwin/64, download the package for the corresponding platform.

After the download is complete, you can extract it without compiling.


#使用 chaosblade
Go to the unzipped folder and you will see the following:
```
├── bin
│ ├── chaos_burncpu
│ ├── chaos_burnio
│ ├── chaos_changedns
│ ├── chaos_delaynetwork
│ ├── chaos_dropnetwork
│ ├── chaos_filldisk
│ ├── chaos_killprocess
│ ├── chaos_lossnetwork
│ ├── jvm.spec.yaml
│ └── tools.jar
├──blade
└── lib
    └── sandbox
```

Where blade is the executable file, the cli of the chaosblade tool, the tool for chaotic experiment execution. Execute `./blade help` to see what support commands are:
```
An easy to use and powerful chaos engineering experiment toolkit

Usage:
  Blade [command]

Available Commands:
  Create Create a chaos engineering experiment
  Destroy destroy a chaos experiment
  Help Help about any command
  Prepare Prepare to experiment
  Revoke Undo chaos engineering experiment preparation
  Status Query preparation stage or experiment status
  Version Print version info

Flags:
  -d, --debug Set client to DEBUG mode
  -h, --help help for blade

Use "blade [command] --help" for more information about a command.
```
## Execute your first chaos experiment
We take the CPU full load (CPU usage is 100%). Example of a walkthrough scenario (!!** Note that if you don't know the influence, don't execute it on the production system machine), execute the following command to implement the experiment:
```
./blade create cpu fullload
```

The execution result returns:
```
{"code":200,"success":true,"result":"7c1f7afc281482c8"}
```

View CPU usage with the `top` command
```
CPU usage: 93.79% user, 6.20% sys, 0.0% idle
```

At this point the command has taken effect, ** stop the chaos experiment**, execute:
```
./blade destroy 7c1f7afc281482c8
```

Return the following results to indicate that the experiment was stopped successfully
```
{"code":200,"success":true,"result":"command: cpu fullload --debug false --help false"}
```

Then look at the CPU and the CPU load has returned to normal:
```
CPU usage: 6.36% user, 4.74% sys, 88.88% idle
```
A CPU full load walkthrough is completed.

## Your second chaotic experiment
In this experiment, we walked through the Dubbo application. Our requirement is that the consumer calls the hello interface under the com.alibaba.demo.HelloService service with a delay of 3 seconds. Next we download the required Dubbo Demo:
 
[dubbo-provider](https://chaosblade.oss-cn-hangzhou.aliyuncs.com/demo/dubbo-provider-1.0-SNAPSHOT.jar)
[dubbo-consumer](https://chaosblade.oss-cn-hangzhou.aliyuncs.com/demo/dubbo-consumer-1.0-SNAPSHOT.jar)

After the download is complete, execute the following command to start the application. Note that you must first start `dubbo-provider` and then start `dubbo-consumer`:
```
# 启动 dubbo-provider
Nohup java -Djava.net.preferIPv4Stack=true -Dproject.name=dubbo-provider -jar dubbo-provider-1.0-SNAPSHOT.jar > provider.nohup.log 2>&1 &

# Wait for 2 seconds, then start dubbo-consumer
Nohup java -Dserver.port=8080 -Djava.net.preferIPv4Stack=true -Dproject.name=dubbo-consumer -jar dubbo-consumer-1.0-SNAPSHOT.jar > consumer.nohup.log 2>&1 &
```
Go to `http://localhost:8080/hello?msg=world` and return the following message, indicating that the startup was successful:
```
{
    Msg: "Dubbo Service: Hello world"
}
```

Next we need to use the blade tool for chaos experiments. Before executing the experiment, we need to execute the prepare command to mount the required java agent:
```
./blade prepare jvm --process dubbo.consumer
```
Returning the following results, indicating that the experiment is ready:
```
{"code":200,"success":true,"result":"e669d57f079a00cc"}
```
We started the chaos experiment. Our requirement is that the consumer calls the `h.allbaba.demo.HelloService` service with a delay of 3 seconds for the `hello` interface.
We execute the `./blade create dubbo delay -h` command to see the command usage of the dubbo call delay:
```
Usage:
  Blade create dubbo delay

Flags:
      --appname string The consumer or provider application name
      --consumer To tag consumer role experiment.
  -h, --help help for delay
      --methodname string The method name in service interface
      --offset string delay offset for the time
      --process string Application process name
      --provider To tag provider experiment
      --service string The service interface
      --time string delay time (required)
      --version string the service version

Global Flags:
  -d, --debug Set client to DEBUG mode
```

Calling the `hello` interface under the `com.alibaba.demo.HelloService` service is delayed for 3 seconds. We execute the following command:
```
./blade create dubbo delay --time 3000 --service com.alibaba.demo.HelloService --methodname hello --consumer --process dubbo.consumer
```
Return the following result, indicating successful execution; access `http://localhost:8080/hello?msg=world` to verify whether the delay is 3 seconds
```
{"code":200,"success":true,"result":"ec695fee1e458fc6"}
```
Parse the commands that implement the experiment:
* `--time`: 3000, indicating a delay of 3000 ms; the unit is ms
* `--service`: com.alibaba.demo.HelloService, indicating the service called
* `--methodname`: hello, indicating the service interface method
* `--consumer`: indicates that the exercise is dubbo consumer
* `--process`: dubbo.consumer, which indicates which application process to implement chaos experiment

Stop the chaotic experiment of the current delay, and access the url again to verify that it returns to normal:
```
./blade destroy ec695fee1e458fc6
```

If you are not happy, we will implement the call to the service just throwing the exception, and execute the `./blade create dubbo throwCustomException -h` command to view the help:
```
Throw custom exception with --exception option

Usage:
  Blade create dubbo throwCustomException

Aliases:
  throwCustomException, tce

Flags:
      --appname string The consumer or provider application name
      --consumer To tag consumer role experiment.
      --exception string Exception class inherit java.lang.Exception (required)
  -h, --help help for throwCustomException
      --methodname string The method name in service interface
      --process string Application process name
      --provider To tag provider experiment
      --service string The service interface
      --version string the service version

Global Flags:
  -d, --debug Set client to DEBUG mode
```
It is similar to the delay command parameter, because the same parameters are needed for the drill dubbo. The difference is that there is no `--time` and a `--exception` parameter.
We simulate calling the service just throwing a `java.lang.Exception` exception:
```
./blade create dubbo throwCustomException --exception java.lang.Exception --service com.alibaba.demo.HelloService --methodname hello --consumer --process dubbo.consumer
```
Return the following results, visit `http://localhost:8080/hello?msg=world` to verify if it is abnormal.
```
{"code":200,"success":true,"result":"09dd96f4c062df69"}
```
Stop the trial and revisit the request to verify that it is restored:
```
./blade destroy 09dd96f4c062df69
```

Finally, we undo the experimental preparation just to uninstall the Java Agent:
```
./blade revoke e669d57f079a00cc
```
If you can't find the UID that was previously returned by prepare, execute the `./blade status --type prepare` command to query:
```
{
        "code": 200,
        "success": true,
        "result": [
                {
                        "Uid": "e669d57f079a00cc",
                        "ProgramType": "jvm",
                        "Process": "dubbo.consumer",
                        "Port": "59688",
                        "Status": "Running",
                        "Error": "",
                        "CreateTime": "2019-03-29T16:19:37.284579975+08:00",
                        "UpdateTime": "2019-03-29T17:05:14.183382945+08:00"
                }
        ]
}

```

# FAQ
### How to get the latest version
Each time chaosblade is released, the relevant changelog and the new version of the package will be synced to RELEASE, which can be downloaded at this address.

### Does the Windows platform have a support plan?
There is no support plan at this time, but you are welcome to mention the issue of related support. The community will decide whether to support according to your needs.

### Execution blade command error: exec format error or cannot execute binary file
Due to the incompatibility between the chaosblade package and the running platform, please let us know [ISSUE] (https://github.com/chaosblade-io/chaosblade/issues) to inform us that the downloaded chaosblade package version and operating system are included in the issue. Version Information.
