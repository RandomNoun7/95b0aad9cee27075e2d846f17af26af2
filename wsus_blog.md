# Combining PowerShell, Bolt and Puppet Tasks - Part 2

In [Part 1](https://puppet.com/blog/combining-powershell-bolt-and-puppet-tasks-part-1) of this blog servies we created and ran a PowerShell script on our local computer, and then turned that into a Bolt Task. We then packaged the module so others could use the task. In Part 2 we going to look at how to test the PowerShell script.

## Why test tasks?

So the most obvious question is, why would I want to test my Puppet Tasks? When we first start writing Tasks it is true that testing isn't really at the front our minds. However if you stop and look at the actions we took in Part1, you can start to see we were _actually_ testing our code, it was just a manual process, for example;

When we wanted to add Task metadata we did the following;

1. Added the metadata information to the `update_history.json` file
2. Ran the `bolt task show` command
3. Verified that the information output from the command (Step 2) was the same as information we added in the metadata file (Step 1)

In software testing this is known as [Arrange, Act, Assert](http://wiki.c2.com/?ArrangeActAssert).  So we can think of what we did as;

* (Arrange) Added the metadata information
* (Act) Ran the bolt command
* (Assert) The output from the command is the same as the metadata information

So this means we're already doing some kind of testing, but it still doesn't answer the question of Why should I test. Testing our tasks means that as we add functionality or change things, we can be sure that it still behaves the same way. And by using an automated testing tool (because let's be honest who likes manual testing anyway!) we can run the tests in our module CI pipeline.

## What testing tools are out there?

While we could use a testing tool to create a Windows Virtual Machine, in the case of the [WSUS Client Module](https://github.com/puppetlabs/puppetlabs-wsus_client) this would be difficult and time consuming to do.  So instead we can use [Pester](https://github.com/pester/Pester) which is a testing and mocking framework for PowerShell.  In fact it's one of only a few Open Source projects which is [shipped with Windows itself!](https://twitter.com/jsnover/status/590178510485860352).

**TODO** Need to add disclaimer.  This isn't how to use pester, but how we integrated pester into testing pipeline

**TODO** Do I need to show how to use Pester?

## Writing testable PowerShell

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/bd92f86c82ebcdf6f93e57490a586cfb68557fbd)

Great, so we can use Pester to test our PowerShell task file, but ... there is a problem.  In order to test the script we need to import it.  We do this using [dot sourcing](https://ss64.com/ps/source.html) the test script.  However this _actually_ runs the script and outputs information.

```
PS> . .\tasks\update_history.ps1
[
    {
        "ServiceID":  "",
        "Title":  "Definition Update for Windows Defender Antivirus - KB2267602 (Definition 1.279.737.0)",
        "UpdateIdentity":  {
                               "RevisionNumber":  200,
                               "UpdateID":  "7cfce973-b755-460c-a1a4-e92512ae2dec"
                           },
        "Categories":  [
                           "Windows Defender"
                       ],
        "Operation":  "Installation",
        "Date":  "2018-10-29 06:55:40Z",
        "ResultCode":  "Succeeded"
    },
    {
...
```

Also, because the script is written with the logic in the root of the script, instead of a function, we have no easy way to execute the script in our tests.

In short, the code I wrote may work, but it was not easily testable!!

### Wrapping the main function

Firstly we need to be able to execution the logic of the script from Pester.  To do this we needed to move all of the logic into it's own function.  For example, if the script used to look like;

```
$Session = New-Object -ComObject "Microsoft.Update.Session"
$Searcher = $Session.CreateUpdateSearcher()
# Returns IUpdateSearcher https://msdn.microsoft.com/en-us/library/windows/desktop/aa386515(v=vs.85).aspx

$historyCount = $Searcher.GetTotalHistoryCount()
if ($historyCount -gt $MaximumUpdates) { $historyCount = $MaximumUpdates }
$Searcher.QueryHistory(0, $historyCount) |
  Where-Object { [String]::IsNullOrEmpty($Title) -or ($_.Title -match $Title) } |
  Where-Object { [String]::IsNullOrEmpty($UpdateID) -or ($_.UpdateIdentity.UpdateID -eq $UpdateID) } |
...
```

We would wrap all of this in a PowerShell function and then call it;

```
Function Invoke-ExecuteTask($Detailed, $Title, $UpdateID, $MaximumUpdates) {
  $Searcher = Get-UpdateSessionObject
  # Returns IUpdateSearcher https://msdn.microsoft.com/en-us/library/windows/desktop/aa386515(v=vs.85).aspx

  $historyCount = $Searcher.GetTotalHistoryCount()
  if ($historyCount -gt $MaximumUpdates) { $historyCount = $MaximumUpdates }
  $Result = $Searcher.QueryHistory(0, $historyCount) |
    Where-Object { [String]::IsNullOrEmpty($Title) -or ($_.Title -match $Title) } |
    Where-Object { [String]::IsNullOrEmpty($UpdateID) -or ($_.UpdateIdentity.UpdateID -eq $UpdateID) } |
  ...
}

Invoke-ExecuteTask -Detailed $Detailed -Title $Title -UpdateID $UpdateID -MaximumUpdates $MaximumUpdates
```

[Full Source Code](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/tasks/update_history.ps1#L68-L110)

Notice how the function `Invoke-ExecuteTask` just wraps around the old logic.  It still does the same thing, just in a function.

Note - For those more advanced in PowerShell you may ask why I didn't use [Cmdlet Binding](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced?view=powershell-6) in the function header. I could have easily defined this is an advanced function however I did not think it was necessary.  The input validation already happens at the top of the script and as this is private function, no user would be explicitly calling it.

### Stopping execution

So now we could call the logic of the script in Pester, but we still had the problem of it actually running the script when we imported it. What we needed was a flag of somekind which could tell the script to execute or not when imported.  There are a number of different types of flags; Setting environment variables or registry keys of files on disk.  However in PowerShell the simpliest method is to just have a script parameter.

Note - Using a script parameter was appropriate for the WSUS Client module but you may prefer to use something else

So what did this look like;

At the top of the script [we added](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/tasks/update_history.ps1#L15-L16) the `NoOperation` parameter;

```
...
  [Parameter(Mandatory = $False)]
  [Switch]$NoOperation
...
```

We also added a simple [if statement](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/tasks/update_history.ps1#L112) at the bottom of the script which conditionally execute the script

```
if (-Not $NoOperation) { Invoke-ExecuteTask }
```

This created a switch parameter called `NoOperation` which would default to `false`, that is, it would execute the script. By using `. .\tasks\update_history.ps1 -NoOperation` we could tell the script to not execute and just import the functions for testing.

## Writing Pester tests

Now that we could successfully import the PowerShell Task file, it was time to write the tests.

### Writing simple tests

The first tests we wrote simply test the enumeration functions, these converted the number style codes into their text version; for example an `OperationResultCode` of 1 means the update is "In Progress"

So first we add the Pester [standard PowerShell commands](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/spec/tasks/update_history.Tests.ps1#L1-L7);
```
$here = Split-Path -Parent $MyInvocation.MyCommand.Path
$sut = (Split-Path -Leaf $MyInvocation.MyCommand.Path).Replace(".Tests.", ".")
$helper = Join-Path (Split-Path -Parent $here) 'spec_helper.ps1'
. $helper
$sut = Join-Path -Path $src -ChildPath "tasks/${sut}"

. $sut -NoOperation
```

These commands;

* Calculate the name of the script being tested (also known as the System Under Test or `$sut`) based on the test file name
* Import any shared helper functions (`spec_helper.ps1`). This blog post didn't add any, but in the future they may be used
* Imports the script under test. Note the use of the new `-NoOperation` parameter

When then test each of the enumeration functions to ensure the the conversions of number to text are what we expect. For example the tests for the [Convert-ToServerSelectionString](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/spec/tasks/update_history.Tests.ps1#L9-L25) function check the output for the numbers 0 to 3

**TODO** Do I need to show a passing pester run?

### Writing more tests

So now we had some simple tests written and passing, we could finish off writing the rest of the tests. Fortunately with testing, we should be describing each of our tests in simple english. I decided that the following tests would be sufficient;

* [should return empty JSON if no history](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L71-L76)
* [should return a JSON array for a single element](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L78-L87)
* [should not return detailed information when Detailed specified as false](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L89-L106)
* [should return detailed information when Detailed specified as true](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L108-L125)
* [should return only the maximum number of updates when specified](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L127-L134)
* [should return a single update when UpdateID is specified](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L136-L145)
* [should return matching updates when Title is specified](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L147-L162)


### Failing tests and mocking

**TODO** Tests were failing due to `f (-Not $NoOperation) { Invoke-ExecuteTask -Detailed $Detailed -Title $Title -U` in https://github.com/puppetlabs/puppetlabs-wsus_client/commit/354c34e19f0f251da6c16c055a137d4fd4a55970

**TODO** Need to talk about mocking `Function Get-UpdateSessionObject() {` in https://github.com/puppetlabs/puppetlabs-wsus_client/commit/354c34e19f0f251da6c16c055a137d4fd4a55970

**TODO** Talk about the fix for https://github.com/puppetlabs/puppetlabs-wsus_client/commit/786717c5dbafa1e33d28d5f2c5a5081d227cf9a8#diff-1b9bad9ab08b53a5e96ced4852470040

---

Changes to Part 1 of the blog

In Part 2, we'll look at how to test our newly created PowerShell Task


