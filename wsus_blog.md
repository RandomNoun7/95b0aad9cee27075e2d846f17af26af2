Theme - Better together story PowerShell + tasks + PE

# Introduce bolt

<TODO Needs a better intro>

For this blog post I'll walk through writing a PowerShell based task for the [WSUS Client Puppet module](https://github.com/puppetlabs/puppetlabs-wsus_client).  This task will enumerate the update history of a computer, so we can see when certain updates have been applied.

# Setting up

## Install Bolt
So first up, we need Puppet Bolt installed.  There are a few ways we could do this on Windows, all of which are describe in the [Bolt documentation](https://puppet.com/docs/bolt/latest/bolt_installing.html).

### Installing Bolt as an MSI

The MSI installers for Bolt are available at [https://downloads.puppet.com/windows/puppet5](https://downloads.puppet.com/windows/puppet5), and the latest installer at [https://downloads.puppet.com/windows/puppet5/puppet-bolt-x64-latest.msi](https://downloads.puppet.com/windows/puppet5/puppet-bolt-x64-latest.msi).  Download the MSI and install through your usual method; command line, right click, SCCM etc.

### Installing Bolt with Chocolatey

Bolt is available as a [chocolatey package](https://chocolatey.org/packages/puppet-bolt).  If you have chocolatey installed, you can use the following command;

``` powershell
PS> choco install puppet-bolt
```

<Do we need to write this?>
### Installing Bolt as a gem

Finally, you can install bolt as a ruby gem.  You could do this directly in your development ruby environment with `gem install bolt`, or by adding it to your Gemfile with `gem "bolt", :require => false`.  Note that if using the Gemfile method, you will need to prefix `bundle exec` for all of the examples below.


## WinRM Configuration

<TODO
Add links to winrmr config in bolt docs

Add links to how to configure winrm from MSDN

Reference docs for winrm quickconfig
>

Bolt can use SSH or WinRM to communicate with nodes, but with Windows a natural choice is WinRM.  While it is outside the scope of this blog to go over how to configure your WinRM Service, for these examples I am using the HTTP Listener (hot recommended for production use), with only the Kerberos and Negotiate authentication methods enabled.  This is a typical configuration when using the `winrm quickconfig` command.

# Running a Hello World bolt command

Now that we have bolt installed we can run a test command to make sure it's working.

``` powershell
PS> bolt command run "Write-Output 'Hello World'" --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Started on 127.0.0.1...
Finished on 127.0.0.1:
  STDOUT:
    Hello World
Successful on 1 node: 127.0.0.1
Ran on 1 node in 2.17 seconds
```

Let's breakdown the command line used

`bolt command run` : This instructs bolt to run a single line command

`"Write-Output 'Hello World'"` : This is the PowerShell command that we will run on the remote computer

`--nodes 127.0.0.1` : We want to run the command against our local computer so we specify the node as 127.0.0.1.  Why not use localhost? As of right now, bolt has an experimental feature for local connections which does yet support PowerShell.  As a workaround we use the loopback address.

`--transport winrm --no-ssl` : We then specify we want bolt to use WinRM, over the HTTP listener (as opposed to the default, HTTPS)

`--user Administrator --password` : We then specify the username as Administrator, and prompt for the password.

The output from bolt shows our Write-Host command:

``` text
  STDOUT:
    Hello World
```

<TODO Get rid of the Hello-World and just use get-process table>

What about something more practical.  Let's list all the running processes;

``` powershell
PS> bolt command run "Get-Process | ConvertTo-JSON" --nodes 127.0.0.1 --transport winrm --no-ssl --user Administrator --password
Please enter your password:
Started on 127.0.0.1...
Finished on 127.0.0.1:
  STDOUT:

...
Lots and lots of text
...

            "UserProcessorTime":  {
                                      "Ticks":  312500,
                                      "Days":  0,
                                      "Hours":  0,
                                      "Milliseconds":  31,
                                      "Minutes":  0,
                                      "Seconds":  0,
                                      "TotalDays":  3.616898148148148E-07,
                                      "TotalHours":  8.6805555555555555E-06,
                                      "TotalMilliseconds":  31.25,
                                      "TotalMinutes":  0.00052083333333333333,
                                      "TotalSeconds":  0.03125
                                  },
            "VirtualMemorySize":  86880256,
            "VirtualMemorySize64":  2203405103104,
            "EnableRaisingEvents":  false,
            "StandardInput":  null,
            "StandardOutput":  null,
            "StandardError":  null,
            "WorkingSet":  7811072,
            "WorkingSet64":  7811072,
            "Site":  null,
            "Container":  null,
            "Name":  "WUDFHost",
            "SI":  0,
            "Handles":  269,
            "VM":  2203405103104,
            "WS":  7811072,
            "PM":  2158592,
            "NPM":  13120,
            "Path":  "C:\\Windows\\System32\\WUDFHost.exe",
            "Company":  "Microsoft Corporation",
            "CPU":  0.078125,
            "FileVersion":  "10.0.17134.1 (WinBuild.160101.0800)",
            "ProductVersion":  "10.0.17134.1",
            "Description":  "Windows Driver Foundation - User-mode Driver Framework Host Process",
            "Product":  "Microsoft® Windows® Operating System",
            "__NounName":  "Process"
        }
    ]
Successful on 1 node: 127.0.0.1
Ran on 1 node in 24.63 seconds
```

Great!  This just listed all of the processes on my machine.  So let's move on to writing more than just a one line command; Puppet Tasks.

# Writing PowerShell tasks

Tasks are similar to PowerShell script files, but they are kept in Puppet Modules and can have metadata. This allows you to reuse and share them more easily.  So the first thing you need when writing a PowerShell task, is a Puppet module.  Tasks reside in the `tasks` directory; For example [here](https://github.com/puppetlabs/puppetlabs-reboot/tree/e81be2b9ad9fa4367648e27e8aea98e3e0ad32dd/tasks) in the Windows Reboot module or [here](https://github.com/puppetlabs/puppetlabs-mysql/tree/bd9d7adcc70be61ca86dcc31e16568e24e96a4bb/tasks) in the MySQL module.

You can create a new task using the [Puppet Development Kit (PDK)](https://puppet.com/docs/pdk/1.x/pdk_reference.html#pdk-new-task-command) using the `pdk new task` command, or by creating a PS1 file in the `tasks` directory.

## Writing a simple task

So let's start with a task in the [WSUS Client module](https://github.com/puppetlabs/puppetlabs-wsus_client) to return the update history of a computer.

We create the file `tasks/update_history.ps1` with the content in [this link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/4bcc461275a50e5bdb88e20fa8ca6ef2dbfd512f) (It's a little long).  You may ask why we don't simply use the popular [PSWindowsUpdate PowerShell module](https://www.powershellgallery.com/packages/PSWindowsUpdate)?  As we'll be running this on remote computers, we don't know if that module is installed, which means we can't use it.

So let's make sure the task exists;

``` powershell
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

* All of the other tasks come as part of Bolt itself.  In this case I'm using Bolt v0.20.5.

* Tasks are named with the name of the module (`wsus_client`) and then the task filename (`update_history`).

So now we know Bolt can see our new task, let's run it;




  ## Where to shove the tasks files (module layout)

  ### PDK or exisiting

  ## Running the PS1 directly


# Run via bolt
  Start with no json tasks file

  Show adding parameters and invocation via bolt

  Show mapping Puppet Types to PowerShell Types (KISS)

  Progression


# Running tasks in PE

  ## getting your module into PE (Forge or Code-Manager)

  ## Running a task on a node with params

  ### Parsing output

  ### Multinode? Node Groups with PQL?  RBAC with tasks?


# What's next

  This example is a read-only getter. what about doing stuff e.g. installing updates

