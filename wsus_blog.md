**TODO Theme - Better together story PowerShell + tasks + PE**
**TODO Should I be using Bolt Task or bolt task?**

# PowerShell and Tasks and Bolt, Oh my! - Part 1
## Writing PowerShell tasks on Windows, for Windows


**TODO Needs a better intro**

For this blog post I'll walk through writing a PowerShell based task for the [WSUS Client Puppet module](https://github.com/puppetlabs/puppetlabs-wsus_client).  This task will enumerate the update history of a computer, so we can see when certain updates have been applied.

**TODO Need to introduce the glossary better, BUT at least I have one!**
## What are Commands, Scripts, Tasks and Plans?

### Commands

A bolt command is a single line of text which can be executed, for example `Write-Host "Hello World!"`

### Scripts

From the [Bolt documentation](https://puppet.com/docs/bolt/latest/running_bolt_commands.html)

> You can execute scripts on remote machines with Bolt.
>
> Bolt copies the script from the local system to the remote node, executes it on that remote node, and then deletes the script from the remote node.

A bolt script is a single file containing many commands.  In this instance it will be a PowerShell file, for example `update_history.ps1`

### Tasks

From the [Bolt documentation](https://puppet.com/docs/bolt/latest/writing_tasks_and_plans.html)

> Puppet tasks are single, ad hoc actions that you can run on target machines in your infrastructure, allowing you to make as-needed changes to remote systems

Tasks use Scripts and then execute them on remote machines.

### Plans

From the [Bolt documentation](https://puppet.com/docs/bolt/latest/writing_tasks_and_plans.html)

> Plans are sets of tasks that can be combined with other logic. This allows you to do more complex task operations, such as running multiple tasks with one command, computing values for the input for a task, or running certain tasks based on results of another task.

# Setting up

## Install Bolt

There are a few ways we could do this on Windows, all of which are described in the [Bolt documentation](https://puppet.com/docs/bolt/latest/bolt_installing.html).

### Installing Bolt as an MSI

The MSI installers for Bolt are available at [https://downloads.puppet.com/windows/puppet5](https://downloads.puppet.com/windows/puppet5), and the latest installer at [https://downloads.puppet.com/windows/puppet5/puppet-bolt-x64-latest.msi](https://downloads.puppet.com/windows/puppet5/puppet-bolt-x64-latest.msi).  Download the MSI and install through your usual method; command line, right click, SCCM etc.

### Installing Bolt with Chocolatey

Bolt is available as a [chocolatey package](https://chocolatey.org/packages/puppet-bolt).  If you have chocolatey installed, you can use the following command;

``` powershell
PS> choco install puppet-bolt
```

### Installing Bolt as a gem

Finally, you can install bolt as a ruby gem.  This is more complicated and the instructions are in the [bolt documentation](https://puppet.com/docs/bolt/latest/bolt_installing.html#task-4877).


## WinRM Configuration

Bolt can use SSH or WinRM to communicate with nodes, but with Windows a natural choice is WinRM.  While it is outside the scope of this blog to go over how to [configure your WinRM Service](https://msdn.microsoft.com/en-us/library/aa384372(v=vs.85).aspx), for these examples I am using the HTTP Listener (not recommended for production use), with only the Kerberos and Negotiate authentication methods enabled.  This is a typical configuration when using the `winrm quickconfig` command.

# Running a single command

Now that we have bolt installed we can run a test command to make sure it's working.  Let's list all the running processes;

``` powershell
PS> bolt command run "Get-Process" --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Started on 127.0.0.1...
Finished on 127.0.0.1:
  STDOUT:

    Handles  NPM(K)    PM(K)      WS(K)     CPU(s)     Id  SI ProcessName
    -------  ------    -----      -----     ------     --  -- -----------
        183       9     3408       1660       0.53   4356   0 AdminService
        790      45    38128      32720     355.83  12396   2 ApplicationFrameHost
        156       8     1724        952       0.05  14296   0 AppVShNotify

...

         72       6     1492       1524       0.08  38212   0 WUDFCompanionHost
        318      14     4668       5000     140.84    500   0 WUDFHost
        674      23    49272      30332   6,460.53  35468   0 WUDFHost

Successful on 1 node: 127.0.0.1
Ran on 1 node in 5.28 seconds
```

Great!  This just listed all of the processes on my machine.  Let's breakdown the command line used

`bolt command run` : In this case we simply want to run a command.  Bolt can also run script files, tasks and plans

`Get-Process` : This is the PowerShell command that we will run on the remote computer

`--nodes 127.0.0.1` : We want to run the command against our local computer so we specify the node as 127.0.0.1.  Why not use localhost? As of right now, bolt has an experimental feature for local connections which does yet support PowerShell.  As a workaround we simply use the loopback address.

`--transport winrm --no-ssl` : We then specify we want bolt to use WinRM, over the HTTP listener (as opposed to HTTPS)

`--user Administrator --password` : We then specify the username as Administrator, and prompt for the password.

While you can add the password on the commandline, it's not very secure as it may appear in your console history or in a tool like [Process Explorer](https://docs.microsoft.com/en-us/sysinternals/downloads/process-explorer), which can see the command line for a process.

It's also a good idea to put `--password` at the very end of the command, so that Bolt doesn't accidentally think other options are the actual password.  For example, don't do this `... --password detailed=true`, as Bolt will try to authenticate with the password `detailed=value` instead of, prompting for the password and then passing the script parameter called `detailed`.

So let's move on to writing more than just a one line command; Puppet Tasks.

# Writing PowerShell tasks

Tasks are similar to PowerShell script files, but they are kept in Puppet Modules and can have metadata. This allows you to reuse and share them more easily.  So the first thing you need when writing a PowerShell task, is a Puppet module.  Tasks reside in the `tasks` directory; For example [here](https://github.com/puppetlabs/puppetlabs-reboot/tree/e81be2b9ad9fa4367648e27e8aea98e3e0ad32dd/tasks) in the Windows Reboot module or [here](https://github.com/puppetlabs/puppetlabs-mysql/tree/bd9d7adcc70be61ca86dcc31e16568e24e96a4bb/tasks) in the MySQL module.

You can create a new task using the [Puppet Development Kit (PDK)](https://puppet.com/docs/pdk/1.x/pdk_reference.html#pdk-new-task-command), using the `pdk new task` command, or by creating a PS1 file in the `tasks` [directory](https://puppet.com/docs/puppet/latest/modules_fundamentals.html#module-structure).

``` text
<MODULE NAME>
  +- manifests
  +- lib
  |    +- facter
  |    ...
  ...
  +- spec
  +- tasks      <---- Tasks go here!
  +- templates
  ...
```

## Creating a PowerShell task

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/4bcc461275a50e5bdb88e20fa8ca6ef2dbfd512f)

So let's start with a task in the [WSUS Client module](https://github.com/puppetlabs/puppetlabs-wsus_client) to return the update history of a computer.

We create the file `tasks/update_history.ps1` with the content in [this link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/4bcc461275a50e5bdb88e20fa8ca6ef2dbfd512f) (It's a little long).  You may ask why we don't simply use the popular [PSWindowsUpdate PowerShell module](https://www.powershellgallery.com/packages/PSWindowsUpdate)?  As we'll be running this on remote computers, we don't know if that module is installed, which means we shouldn't use it.  This is important to remember if you later publish your module for other people to use.

One of the great things about having a script file is that we can use our normal PowerShell tools (VS Code, ISE etc.) to write and test the script, and then use bolt to execute it.

To run the task manually, we can use normal PowerShell commands;

``` text
PS> .\tasks\update_history.ps1
    {
        "ServerSelection":  "WindowsUpdate",
        "ClientApplicationID":  "Windows Defender Antivirus (77BDAF73-B396-481F-9042-AD358843EC24)",
        "Categories":  [
                           "Windows Defender"
                       ],
        "UnmappedResultCode":  0,
        "Title":  "Definition Update for Windows Defender Antivirus - KB2267602 (Definition 1.269.1089.0)",
        "UpdateIdentity":  {
                               "RevisionNumber":  200,
                               "UpdateID":  "340544b8-e4f0-4eb3-b7d8-04e6608986ea"
                           },
        "UninstallationNotes":  "",
        "Description":  "Install this update to revise the definition files that are used to detect viruses, spyware, and other potentially unwanted software. Once you have installed this item, it cannot be removed.",
        "SupportUrl":  "http://go.microsoft.com/fwlink/?LinkId=52661",
        "ServiceID":  "",
        "UninstallationSteps":  [

                                ],
        "Operation":  "Installation",
        "Date":  "2018-06-12 01:48:16Z",
        "ResultCode":  "Succeeded",
        "HResult":  0
    },
...
    {
        "ServerSelection":  "Other",
        "ClientApplicationID":  "UpdateOrchestrator",
        "Categories":  [

                       ],
        "UnmappedResultCode":  0,
        "Title":  "Feature update to Windows 10, version 1803",
        "UpdateIdentity":  {
                               "RevisionNumber":  1,
                               "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                           },
        "UninstallationNotes":  "",
        "Description":  "Install the latest update for Windows 10: the Windows 10 April 2018 Update.",
        "SupportUrl":  "",
        "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
        "UninstallationSteps":  [

                                ],
        "Operation":  "Installation",
        "Date":  "2018-05-02 01:41:20Z",
        "ResultCode":  "Succeeded",
        "HResult":  0
    }
]
```

# Running a PowerShell task

Now we can use bolt to run the task remotely.  But first let's make sure the task exists;

``` text
PS> bolt task show --modulepath modules

apply::resource               Apply a single Puppet resource
facts                         Gather system facts
facts::bash
facts::powershell
facts::ruby
package                       Manage and inspect the state of packages
puppet_conf                   Inspect puppet agent configuration settings
service                       Manage and inspect the state of services
service::linux                Manage the state of services (without a puppet agent)
service::windows              Manage the state of Windows services (without a puppet agent)
wsus_client::update_history
```

Let's breakdown the command line used

`bolt task show` : This instructs bolt to list all of the tasks it knows about

`--modulepath C:\modules` : As tasks are located in Puppet modules we need to tell bolt where the modules are located.  In this case my modules are located in `C:\modules`, and the WSUS Client module is at `C:\modules\wsus_client`.

The output shows lots of task names with our new task down the bottom of the list.

```
wsus_client::update_history
```

* All of the other tasks come as part of Bolt itself.  In this instance, we're using Bolt v0.20.5.

* Tasks are uniquely named by the name of the module (`wsus_client`), a double colon (`::`) and then the task filename (`update_history`)

So now we know Bolt can see our new task, let's run it;

``` powershell
PS> bolt task run wsus_client::update_history --modulepath modules --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Started on 127.0.0.1...
Finished on 127.0.0.1:
  [
      {
          "ServerSelection":  "WindowsUpdate",
          "ClientApplicationID":  "Windows Defender Antivirus (77BDAF73-B396-481F-9042-AD358843EC24)",
                             "Windows Defender"
          "Categories":  [
                         ],
          "UnmappedResultCode":  0,
          "Title":  "Definition Update for Windows Defender Antivirus - KB2267602 (Definition 1.269.1089.0)",
          "UpdateIdentity":  {
                                 "RevisionNumber":  200,

                                 "UpdateID":  "340544b8-e4f0-4eb3-b7d8-04e6608986ea"

                             },
          "UninstallationNotes":  "",
          "Description":  "Install this update to revise the definition files that are used to detect viruses, spyware, and other potentially unwanted software. Once you have installed this item, it cannot be removed.",
          "SupportUrl":  "http://go.microsoft.com/fwlink/?LinkId=52661",
          "ServiceID":  "",
          "UninstallationSteps":  [
                                  ],
          "Operation":  "Installation",
          "Date":  "2018-06-12 01:48:16Z",
          "ResultCode":  "Succeeded",
          "HResult":  0
      },
...
      {
          "ServerSelection":  "Other",
          "UninstallationSteps":  [
          "ClientApplicationID":  "UpdateOrchestrator",
          "Categories":  [
                         ],
          "UnmappedResultCode":  0,
          "Title":  "Feature update to Windows 10, version 1803",
          "UpdateIdentity":  {
                                 "RevisionNumber":  1,
                                 "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                             },
          "UninstallationNotes":  "",
          "Description":  "Install the latest update for Windows 10: the Windows 10 April 2018 Update.",
          "UninstallationSteps":  [
          "SupportUrl":  "",
          "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
          "Date":  "2018-05-02 01:41:20Z",
          "Operation":  "Installation",
          "ResultCode":  "Succeeded",
                                  ],
      }
          "HResult":  0
  ]
  {
  }
Successful on 1 node: 127.0.0.1
Ran on 1 node in 12.43 seconds
```

Comparing the output of the manual process versus the bolt process, they look almost the same.  There's additional data added at the end of the bolt output.

```
...
{
}
```

This will have error information when the task fails to run. **TODO Need to confirm this**

## Why use ConvertTo-JSON?

You may have noticed that the output from the script is not pure text, but is JSON encoded text.  This comes from the [last line in the PowerShell script](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/5eff63e891822d4efa8c1af29940b66fa5b9aee3/tasks/update_history.ps1#L105)

``` powershell
} | ConvertTo-JSON
```

Bolt tasks return text, but if we want the output of the task to be used by other tools or processes, the output should be structured text.  In particular, the output can be used by [Bolt Plans](https://puppet.com/docs/bolt/0.x/writing_plans.html) which can orchestrate multiple Bolt Tasks.  Bolt uses [JSON structured text]((https://puppet.com/docs/bolt/0.x/writing_tasks.html#concept-87)) for it's structured output format, which is great as PowerShell has native support for JSON, through the `ConvertTo-JSON` function in PowerShell 3.0 and above.

So what about PowerShell 2.0? Right now you would need to output the equivalent text by yourself in the PowerShell script; For example;

``` powershell
PS> $value = 'This is some text'; Write-Output "{ `"output`": `"${value}`"}"
{ "output": "This is some text"}
```

The string is now in a JSON format.

For small, simple PowerShell scripts this method may be fine, but for complex data, like the `update_history.ps1` file we used earlier, it's quite difficult to do.  A quick search in your favourite search engine for "convertto-json powershell 2" may guide you to some good workarounds.

There is a [feature request](https://tickets.puppetlabs.com/browse/BOLT-406) to add JSON support for PowerShell 2.0, but right now that is not available.

# Adding Script Parameters

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/f2ade674807c8079d97158adacda599f0d67b345)

The output from the script is quite verbose.  What we really want is only return the information we need, but still have the ability to get everything.  So we need to add a `Detailed` script parameter;

For brief information

``` powershell
PS> .\tasks\update_history.ps1
```

And for detailed information

``` powershell
PS> .\tasks\update_history.ps1 -Detailed
```

Bolt supports passing parameters to PowerShell scripts through named parameters.  So we need to add [cmdlet binding](https://puppet.com/docs/bolt/latest/writing_tasks.html#defining-parameters-in-windows) to the top of our script and specify the Detailed parameter.

``` powershell
[CmdletBinding()]
Param(
  [Parameter(Mandatory = $False)]
  [Switch]$Detailed
)
...
```

And then change our output to add the additional settings.  I've left this out of the blog post but you can see them on the [WSUS Client GitHub repository](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/f2ade674807c8079d97158adacda599f0d67b345#diff-1b9bad9ab08b53a5e96ced4852470040)

So again let's try this locally in PowerShell;

``` text
PS> .\tasks\update_history.ps1
...
    {
        "Categories":  [

                       ],
        "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
        "UpdateIdentity":  {
                               "RevisionNumber":  1,
                               "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                           },
        "Date":  "2018-05-02 01:41:20Z",
        "ResultCode":  "Succeeded",
        "Operation":  "Installation",
        "Title":  "Feature update to Windows 10, version 1803"
    }
]
```

``` text
PS> .\tasks\update_history.ps1 -Detailed
...
    {
        "ServerSelection":  "Other",
        "ClientApplicationID":  "UpdateOrchestrator",
        "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
        "Title":  "Feature update to Windows 10, version 1803",
        "UnmappedResultCode":  0,
        "UpdateIdentity":  {
                               "RevisionNumber":  1,
                               "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                           },
        "UninstallationNotes":  "",
        "Description":  "Install the latest update for Windows 10: the Windows 10 April 2018 Update.",
        "SupportUrl":  "",
        "Categories":  [

                       ],
        "Operation":  "Installation",
        "Date":  "2018-05-02 01:41:20Z",
        "ResultCode":  "Succeeded",
        "HResult":  0,
        "UninstallationSteps":  [

                                ]
    }
]
```

Great! We can now change how much information we return, but how does Bolt use this? Bolt uses a metadata file to store information about the task, including the available parameters and their type.

# Adding Task metadata

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/f2ade674807c8079d97158adacda599f0d67b345#diff-bba989e98a0c7d73b6871b603f9e81a8)

[Task metadata files](https://puppet.com/docs/bolt/latest/writing_tasks.html#concept-677) are JSON formatted files with the same name as their script.  This means with our script called `tasks\update_history.ps1`, the metadata file will be called `tasks\update_history.json`.  So let's create that file with information about our task;

``` json
{
  "description": "Returns a history of installed Windows Updates.",
  "parameters": {
    "detailed": {
      "description": "Return detailed update information.  Default is to return basic information",
      "type": "Optional[Boolean]"
    }
  },
  "input_method": "powershell"
}
```

Let's break this down;

`"description": "Returns a history of installed Windows Updates.",` : This is a short description of the task.  When we previously ran the `bolt task show` command you may have noticed there's a description column, and some tasks had information there.  This is where that information comes from.

`"parameters": {` : This is where we define the new Detailed parameter

`"detailed": {` : The is the name of the new parameter.  Note that it is in lower case whereas in the script it's mixed case

`"description": "Return detailed update ...,` : This is a short description of the parameter and is useful for people to understand how your task works

`"type": "Optional[Boolean]"` : This defines the type of data we expect from the user when running the task and whether it is mandatory or optional.  Bolt types will be looked at next. **TODO Don't like this reference but not sure what else to use**

`"input_method": "powershell"` : This tells bolt that it should use the PowerShell method when sending script parameters.  Normally this is not required as PowerShell script files (.PS1) will automatically use this method.  However at the time of writing there is a [bug in Bolt](https://tickets.puppetlabs.com/browse/BOLT-536), and as a workaround the input method has to be defined.

The [bolt documentation](https://puppet.com/docs/bolt/latest/writing_tasks.html#reference-9297) lists all of the available settings in the metadata file.


### Choosing a bolt parameter type

In our PowerShell script the `Detailed` parameter is defined as;

```powershell
  [Parameter(Mandatory = $False)]
  [Switch]$Detailed
```

This is a boolean parameter which is not mandatory.  The equivalent defintion in a Bolt type is;

``` text
Optional[Boolean]
```

This reads as a boolean type which is optional; that is, not mandatory.

The [Bolt parameter types](https://puppet.com/docs/bolt/latest/writing_tasks.html#reference-3806) come from the Puppet Type system and can, mostly, be directly translated into PowerShell types and PowerShell parameter attributes

| Bolt Type            | PowerShell Parameter |
| -------------------- | -------------------- |
| `String`             | `[Parameter(Mandatory = $True)] [String] $Param` |
| `Optional[String]`   | `[String] $Param` |
| `String[5]`          | `[Parameter(Mandatory = $True)] [ValidateLength(5)] [String] $Param` |
| `Pattern[/\A\w+\Z/]` | `[Parameter(Mandatory = $True)] [ValidatePattern({\A\w+\Z})] [String] $Param` |
| `Integer`            | `[Parameter(Mandatory = $True)] [Int] $Param` |
| `Integer[1, 20]`     | `[Parameter(Mandatory = $True)] [ValidateRange(1, 20)] [Int] $Param` |
| `Optional[Integer]`  | `[Int] $Param` |
| `Boolean`            | `[Parameter(Mandatory = $True)] [Switch] $Param` |
| `Boolean`            | `[Parameter(Mandatory = $True)] [Bool] $Param` |
| `Optional[Boolean]`  | `[Switch] $Param` |
| `Optional[Boolean]`  | `[Bool] $Param` |

* This is not a complete list, but commonly used script parameters

* In Bolt, all parameters are mandatory unless the `Optional[]` type is used, whereas in PowerShell, parameters are optional unless `Mandatory = $True` is set

* The default values of a task parameter need to be set in the PowerShell script, but are generally documented in the task matadata file

* While you can create complex Bolt types and PowerShell parameters, it would be best to keep them as simple as possible (String, Int, Boolean) as the translation between both types is not always exact.  For example PowerShell parameters can use [Position](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?view=powershell-6#position-argument), [ParameterSetName](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?view=powershell-6#parametersetname-argument) and [ValidateScript](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced_parameters?view=powershell-6#validatescript-validation-attribute), but they have no comparable Bolt type.

The full list of available parameter types is located in the [Bolt documentation](https://puppet.com/docs/bolt/latest/writing_tasks.html#reference-3806), and more detailed Puppet type information in the [Puppet documentation](https://puppet.com/docs/puppet/latest/lang_data_type.html).

## Viewing Task metadata

Now that we have some task metadata, let's display that information in bolt;

``` text
PS> bolt task show --modulepath modules

...
wsus_client::update_history   Returns a history of installed Windows Updates.
```

We can now see the description of task in the output.  Now let's get more more information about our task;

``` text
PS> bolt task show wsus_client::update_history --modulepath modules

wsus_client::update_history - Returns a history of installed Windows Updates.

USAGE:
bolt task run --nodes, -n <node-name> wsus_client::update_history [detailed=<value>]

PARAMETERS:
- detailed: Optional[Boolean]
    Return detailed update information.  Default is to return basic information
```

By adding the task name to the show command (`... show show wsus_client::update_history `) the output shows the complete information about the task, including all available parameters.

# Running a task with parameters

The `task show` Bolt command actaully gives us an example of how to use task parameters `... [detailed=<value>]`.  So let's run the task with detailed output;

``` text
PS> bolt task run wsus_client::update_history detailed=true --modulepath modules --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Started on 127.0.0.1...
Finished on 127.0.0.1:
...
      },
      {
          "ServerSelection":  "Other",
          "ClientApplicationID":  "UpdateOrchestrator",
          "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
          "Title":  "Feature update to Windows 10, version 1803",
          "UnmappedResultCode":  0,
          "UpdateIdentity":  {
                                 "RevisionNumber":  1,
                                 "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                             },
          "UninstallationNotes":  "",
          "Description":  "Install the latest update for Windows 10: the Windows 10 April 2018 Update.",
          "SupportUrl":  "",
          "Categories":  [

                         ],
          "Operation":  "Installation",
          "Date":  "2018-05-02 01:41:20Z",
          "ResultCode":  "Succeeded",
          "HResult":  0,
          "UninstallationSteps":  [

                                  ]
      }
  ]
  {
  }
Successful on 1 node: 127.0.0.1
Ran on 1 node in 3.14 seconds
```

We added `detailed=true` to the command line, which passes the parameter to the PowerShell script.  We can also use `detailed=false` to return only the basic information, which is the same as the default behavior;

``` text
PS> bolt task run wsus_client::update_history detailed=false --modulepath modules --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Started on 127.0.0.1...
Finished on 127.0.0.1:
...
      },
      {
          "Categories":  [

                         ],
          "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
          "UpdateIdentity":  {
                                 "RevisionNumber":  1,
                                 "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                             },
          "Date":  "2018-05-02 01:41:20Z",
          "ResultCode":  "Succeeded",
          "Operation":  "Installation",
          "Title":  "Feature update to Windows 10, version 1803"
      }
  ]
  {
  }
Successful on 1 node: 127.0.0.1
Ran on 1 node in 2.65 seconds
```

What if we pass in something other than `true` or `false`? such as `abc123`?

``` text
PS> bolt task run wsus_client::update_history detailed=abc123 --modulepath modules --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Task wsus_client::update_history:
 parameter 'detailed' expects a value of type Undef or Boolean, got String
```

Bolt will validate the input, but as we'll see later, it's really useful when used with Puppet Enterprise.

# Adding more parameters

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/5eff63e891822d4efa8c1af29940b66fa5b9aee3)

Instead of returning all of the updates, it would be great to add some filtering, by their name, by a unique identification number (UpdateID) and to limit the total number returned.  Let's add three more parameters using the same development process.

The parameters we'll add are;

`title` : Return updates which match the specified regular expression.  Default is to all updates

`updateid` : Return updates which the specified Update ID.  Default is to all update

`maximumupdates` : Limit the size of the history returned.  Default is to return a maximum of 300 items

And the development process;

1. Add the parameters to the PowerShell file
2. Make changes to the PowerShell file and test locally
3. Add the parameters to the task metadata
4. Test the metadata changes can be seen by Bolt
5. Run the task with the new parameters using Bolt


### 1. Add the parameters to the PowerShell file

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/5eff63e891822d4efa8c1af29940b66fa5b9aee3/tasks/update_history.ps1#L6-L14)

``` powershell
  [Parameter(Mandatory = $False)]
  [String]$Title,

  [Parameter(Mandatory = $False)]
  [String]$UpdateID,

  [Parameter(Mandatory = $False)]
  [Int]$MaximumUpdates = 300
```

### 2. Make changes to the PowerShell file and test locally

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/5eff63e891822d4efa8c1af29940b66fa5b9aee3/tasks/update_history.ps1#L70-L73)

``` text
PS> get-help .\tasks\update_history.ps1
update_history.ps1 [[-Title] <string>] [[-UpdateID] <string>] [[-MaximumUpdates] <int>] [-Detailed] [<CommonParameters>]

PS> .\tasks\update_history.ps1 -UpdateID 6850722b-d202-417f-b6d3-f45419191852
{
    "Categories":  [

                   ],
    "ServiceID":  "8b24b027-1dee-babb-9a95-3517dfb9c552",
    "UpdateIdentity":  {
                           "RevisionNumber":  1,
                           "UpdateID":  "6850722b-d202-417f-b6d3-f45419191852"
                       },
    "Date":  "2018-05-02 01:41:20Z",
    "ResultCode":  "Succeeded",
    "Operation":  "Installation",
    "Title":  "Feature update to Windows 10, version 1803"
}
```

### 3. Add the parameters to the task metadata

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/5eff63e891822d4efa8c1af29940b66fa5b9aee3/tasks/update_history.json#L8-L19)

``` json
    "title": {
      "description": "Return updates which match the specified regular expression.  Default is to all updates",
      "type": "Optional[String]"
    },
    "updateid": {
      "description": "Return updates which the specified Update ID.  Default is to all updates",
      "type": "Optional[String]"
    },
    "maximumupdates": {
      "description": "Limit the size of the history returned.  Default is to return a maximum of 300 items",
      "type": "Optional[String]"
    }
```

Note, I should not have used `Optional[String]` for the `maximumupdates` parameter.  It really should have been `Optional[Integer[0]]`.

### 4. Test the metadata changes can be seen by Bolt

``` text
PS> bolt task show wsus_client::update_history --modulepath modules

wsus_client::update_history - Returns a history of installed Windows Updates.

USAGE:
bolt task run --nodes, -n <node-name> wsus_client::update_history [detailed=<value>] [title=<value>] [updateid=<value>] [maximumupdates=<value>]

PARAMETERS:
- detailed: Optional[Boolean]
    Return detailed update information.  Default is to return basic information
- title: Optional[String]
    Return updates which match the specified regular expression.  Default is to all updates
- updateid: Optional[String]
    Return updates which the specified Update ID.  Default is to all updates
- maximumupdates: Optional[String]
    Limit the size of the history returned.  Default is to return a maximum of 300 items
```

### 5. Run the task with the new parameters using Bolt

``` text
PS> bolt task run wsus_client::update_history updateid=6850722b-d202-417f-b6d3-f45419191852 --modulepath modules --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Started on 127.0.0.1...
Finished on 127.0.0.1:
  {
    "Categories": [

    ],
    "ServiceID": "8b24b027-1dee-babb-9a95-3517dfb9c552",
    "UpdateIdentity": {
      "RevisionNumber": 1,
      "UpdateID": "6850722b-d202-417f-b6d3-f45419191852"
    },
    "Date": "2018-05-02 01:41:20Z",
    "ResultCode": "Succeeded",
    "Operation": "Installation",
    "Title": "Feature update to Windows 10, version 1803"
  }
Successful on 1 node: 127.0.0.1
Ran on 1 node in 3.39 seconds
```

# Packaging the module

Now that we have our task working we can share the module, and the task, on the [Puppet Forge](https://forge.puppet.com/), or an internal repository.  How to package and publish your module is described in the [PDK](https://puppet.com/docs/pdk/latest/pdk_building_module_packages.html) and [Puppet](https://puppet.com/docs/puppet/latest/bgtm.html#releasing-your-module) documentation.

The Puppet Forge also includes a helpful [search filter](https://forge.puppet.com/modules?utf-8=%E2%9C%93&page_size=25&with_tasks=true) to find modules that have tasks.


---------------------------------------------------------
**TODO Add wrap up an segway into Part 2**

Perhaps a what's next. Adding a task which does stuff instead of getting stuff

---------------------------------------------------------


---------------------------------------------------------
**Break here for Part 2**
---------------------------------------------------------



# YET TO BE WRITTEN


  Progression - adding more stuff


# Running tasks in PE

  ## Running a task on a node with params

  ### Parsing output

  ### Multinode? Node Groups with PQL?  RBAC with tasks?


# What's next

  This example is a read-only getter. what about doing stuff e.g. installing updates

