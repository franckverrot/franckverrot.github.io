---
title: "Introducing Trek"
date: 2019-01-21 21:02:12
slug: "introducing-trek"
tags:
  - go
  - open-source
  - infrastructure
---


Having spent a lot of time working with the Hashicorp stack lately,
I have been working a lot with the HashiCorp stack lately, mostly with
Consul, Nomad, and soon Vault.  Even if I was more used to operating
Kubernetes, I really appreciate the simplicity and focus that HashiCorp
builds into its products.

I also spend a lot of time in the console (mix of tmux, vim – or Visual Studio Code when pairing with people – and other CLI tools), so I wanted to find a tool that would keep me in the shell, and I eventually released it.  Today, I'd like to introduce you to [Trek][trek].

<!--more-->

Trek is a CLI tool that allows you to:

  * navigate your Nomad clusters
  * format your search results for easy integrations with your own tools


### Navigating Nomad clusters

```
trek -ui=true
```

![preview](https://raw.githubusercontent.com/franckverrot/trek/master/assets/jan-15-screenshot.png)

<center>Screenshot from the CLI</center>

After writing [a simple configuration file][config_file] (it works without
one with a local setup), you should be able to navigate (arrow keys) your
clusters to discover your jobs, task groups, allocations and tasks. An extra
dialog with more information about your tasks is also available.


### Formatting search results

Trek is using Go's standard formatter library [`fmt`][fmt] to format its search results, which makes it super convenient to pipe the output of `trek` into other tools.

```
# Where is my task `redis6` running?
λ trek -job example34 -task-group cache56 -allocation 0 -task redis6 -display-format "{{range .Network.Ports}}{{$.IP}}:{{.Value}}{{println}}{{end}}"
127.0.0.1:31478

# Let's go there
λ ssh $(trek -job example34 -task-group cache56 -allocation 0 -task redis6 -display-format "{{range .Network.Ports}}{{$.IP}}:{{.Value}}{{println}}{{end}}")
```


### How to get it

[A binary has been released][binary], only works on MacOS for now but will release more in the next releases as I also started using it on Linux more frequently.

Please give it a try, and send feedbacks!



[trek]: https://github.com/franckverrot/trek
[config_file]: https://github.com/franckverrot/trek#trek-configuration-file
[fmt]: https://golang.org/pkg/fmt/
[binary]: https://github.com/franckverrot/trek/releases