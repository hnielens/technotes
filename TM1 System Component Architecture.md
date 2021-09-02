## Start and stop Planning Analytics TM1 Services on Linux

Start Admin Server
```
{{install_dir}}/bin64/startup_tm1admsrv.sh
```

Start a TM1 Server manually:
```
{{install_dir}}/bin64/startup_tm1s.sh ../samples/tm1/PlanSamp
```

Stop a server manually
```
{{install_dir}}/bin64/tm1srvstop.exe -n "Planning Sample" -v 10.68.178.33 -user admin -pwd apple
```

Start Admin Server
```
{{install_dir}}/bin64/startup_tm1admsrv.sh
```

Start the PA Administration Agent
```
{{install_dir}}/paa_agent/bin/startup_agent.sh
```

Stop the PA Administration Agent
```
{{install_dir}}/paa_agent/bin/shutdown_agent.sh
```

## Show PA TM1 Related Processes

Show all PA TM1 processes
```
ps -ef | grep tm1 | grep -v grep
```
Show the PA TM1 Admin Server process
```
ps -ef | grep tm1admsrv | grep -v grep

```
Show the PA Administration Agent
```
ps -ef | grep paa_agent | grep -v grep
```

Show the processes related to PA TM1 Server
```
ps -ef | grep tm1s.exe | grep -v grep

```
