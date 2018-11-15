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

While we could use a testing tool to create a Windows Virtual Machine, in the case of the [WSUS Client Module](https://github.com/puppetlabs/puppetlabs-wsus_client) this would be difficult and time consuming to do.  So instead we can use [Pester](https://github.com/pester/Pester) which is a testing and mocking framework for PowerShell.  In fact it's one of only a few Open Source projects which is [shipped in Windows itself!](https://twitter.com/jsnover/status/590178510485860352).

Note - this blog post won't go into how to write Pester tests.  A quick [search](https://www.google.com.au/search?q=how+to+write+Pester+tests) will help you, as well as some great talks by [Jakub Jares](https://www.youtube.com/watch?v=5OtGvm7HMMs) and myself, [Glenn Sarti](https://www.youtube.com/watch?v=NrUxgSaFvtk).


**TODO** Do I need to show how to use Pester? Probably not

## Writing testable PowerShell

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/bd92f86c82ebcdf6f93e57490a586cfb68557fbd)

Great, so we can use Pester to test our PowerShell task file, but ... there is a problem.  In order to test the script we need to import it.  We do this by [dot sourcing](https://ss64.com/ps/source.html) the test script.  However this _actually_ runs the script and outputs information.

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

Also, because the script is written with the logic in the root, instead of in a function, we have no easy way to execute the script in our tests.

In short, the code I wrote may work, but it was not easily testable!

### Wrapping the main function

Firstly we need to be able to separate loading the script and running the script. To do this we needed to move all of the logic into it's own function. For example, if the script used to look like;

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

Note - For those more advanced in PowerShell you may ask why I didn't use [Cmdlet Binding](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions_advanced?view=powershell-6) in the function header. I could have easily defined this is an advanced function however I did not think it was necessary.  The input validation already happens at the top of the script and as this is private function, and no user would be explicitly calling it.

### Stopping execution

So now we could call the logic of the script in Pester, but we still had the problem of it actually running the script when we imported it. What we needed was a flag of some kind which could tell the script to execute or not when imported.  There are a number of different types of flags; Setting environment variables or registry keys of files on disk.  However in PowerShell the simpliest method is to just have a script parameter.

Note - Using a script parameter was appropriate for the WSUS Client module but you may prefer to use something else

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

Note - For those more advanced in PowerShell you may ask why I didn't use the [WhatIf](https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_commonparameters?view=powershell-6#risk-management-parameter-descriptions) parameter instead. The `WhatIf` parameter is more geared towards a user interaction. While yes it could've been used, all we needed was just simple switch parameter.

## Writing Pester tests

Now that we could successfully import the PowerShell Task file, it was time to write the tests.

### Writing simple tests

The first tests simply tested the enumeration functions. These functions converted the number style codes into their text version; for example an `OperationResultCode` of 1 means the update is "In Progress"

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

So now we had some simple tests written, and passing, we could finish off writing the rest of the tests. Fortunately with testing, we should be describing each of our tests in simple english. I decided that the following tests would be sufficient;

* [should return empty JSON if no history](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L71-L76)
* [should return a JSON array for a single element](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L78-L87)
* [should not return detailed information when Detailed specified as false](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L89-L106)
* [should return detailed information when Detailed specified as true](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L108-L125)
* [should return only the maximum number of updates when specified](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L127-L134)
* [should return a single update when UpdateID is specified](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L136-L145)
* [should return matching updates when Title is specified](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L147-L162)

### More testing issues

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/354c34e19f0f251da6c16c055a137d4fd4a55970)

While writing the test it became apparent that the `Invoke-ExecuteTask` function still wasn't easily testable. The function created a[`Microsoft.Update.Session`](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/tasks/update_history.ps1#L69-L71) COM object.  This object was then used to query the system for update history.  However this meant the testing could only query the existing system, and that we wouldn't be able to see the behaviour if there no updates available, or 1000 updates.  What we needed to do was mock the respsonse of the COM object so we could test the function properly.

Fortunately Pester provides a mocking feature, however the function needed to be modified so we could mock the response.  So again we wrapped the logic in another function:

Previously we [had](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/bd92f86c82ebcdf6f93e57490a586cfb68557fbd/tasks/update_history.ps1#L68-L71)
``` powershell
Function Invoke-ExecuteTask() {
  $Session = New-Object -ComObject "Microsoft.Update.Session"
  $Searcher = $Session.CreateUpdateSearcher()
  # Returns IUpdateSearcher https://msdn.microsoft.com/en-us/library/windows/desktop/aa386515(v=vs.85).aspx

...
```

and after [wrapping the object creation](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/tasks/update_history.ps1#L68-L75)

``` powershell
Function Get-UpdateSessionObject() {
  $Session = New-Object -ComObject "Microsoft.Update.Session"
  Write-Output $Session.CreateUpdateSearcher()
}

Function Invoke-ExecuteTask($Detailed, $Title, $UpdateID, $MaximumUpdates) {
  $Searcher = Get-UpdateSessionObject
  # Returns IUpdateSearcher https://msdn.microsoft.com/en-us/library/windows/desktop/aa386515(v=vs.85).aspx

...
```

Now we could mock the response from `Get-UpdateSessionObject` to simulate any number or kind of updates with the testing helpers [`New-MockUpdateSession`](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/spec_helper.ps1#L33-L50) and [`New-MockUpdate`](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/spec_helper.ps1#L5-L32).

For example, the [`should return empty JSON if no history`](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/354c34e19f0f251da6c16c055a137d4fd4a55970/spec/tasks/update_history.Tests.ps1#L71-L76) test mocks an update session with no updates, using the Pester `Mock` function;

``` powershell
  Mock Get-UpdateSessionObject { New-MockUpdateSession 0 }
```

### Failing tests

[Source Code Link](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/786717c5dbafa1e33d28d5f2c5a5081d227cf9a8)

Running the Pester tests showed a failure.  The `should return a JSON array for a single element` was failing;

``` text
  Describing Invoke-ExecuteTask
    [+] should return empty JSON if no history 472ms
    [-] should return a JSON array for a single element 162ms
      Expected regular expression '^\[' to match '{
          "Categories":  [
                         ],
          "ServiceID":  "d605c6f0-cdea-4b1e-a225-e643254056d4",
          "UpdateIdentity":  {
                                 "RevisionNumber":  3,
                                 "UpdateID":  "d306e6b6-dd95-46ed-be96-137ecddd8611"
                             },
          "Date":  "2018-11-15 14:30:15Z",
          "ResultCode":  "Succeeded With Errors",
          "Operation":  "Uninstallation",
          "Title":  "Mock Update Title 1724034957"
      }', but it did not match.
      82:     $ResultJSON | Should -Match "^\["
      at <ScriptBlock>, C:\Source\puppetlabs-wsus_client\spec\tasks\update_history.Tests.ps1: line 82
    [+] should not return detailed information when Detailed specified as false 156ms
    [+] should return detailed information when Detailed specified as true 74ms
    [+] should return only the maximum number of updates when specified 73ms
    [+] should return a single update when UpdateID is specified 71ms
    [+] should return a matching updates when Title is specified 73ms
```

This failure turned out to be a valid.  When the bolt task runs it should return a JSON Array, even for a single update.  This turned out to be a percularity with PowerShell and piping objects.  With a single object in the pipe, the JSON conversion just returns the object, whereas with two or more objects the JSON convertsion returns an array.

In this case the [fix](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/786717c5dbafa1e33d28d5f2c5a5081d227cf9a8) was fairly simple.  I manually added the opening and closing brackets to the string if there was only one object in the pipe! Running Pester again showed all tests passed!

``` text
  Describing Invoke-ExecuteTask
    [+] should return empty JSON if no history 42ms
    [+] should return a JSON array for a single element 60ms
    [+] should not return detailed information when Detailed specified as false 99ms
    [+] should return detailed information when Detailed specified as true 61ms
    [+] should return only the maximum number of updates when specified 68ms
    [+] should return a single update when UpdateID is specified 31ms
    [+] should return a matching updates when Title is specified 32ms
```

## Running tests automatically

[Source Code Link #1](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/e62e2691d266d797cc648a8975a69f411c70b03a)

[Source Code Link #2](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/1039ebf83c4575c419aa4b70178b8afdad0da2a3)

Having a suite of tests to run was nice, but we really needed them to be run in a Continuous Integration (CI) pipeline.  Fortunately the WSUS_Client module was already setup with an [AppVeyor](https://www.appveyor.com/) [CI pipeline](https://ci.appveyor.com/project/puppetlabs/puppetlabs-wsus-client);

* I added a [small helper script](https://github.com/puppetlabs/puppetlabs-wsus_client/commit/e62e2691d266d797cc648a8975a69f411c70b03a) to install Pester if it didn't exist and then actually run Pester

* I modified [the Rakefile](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/1039ebf83c4575c419aa4b70178b8afdad0da2a3/Rakefile#L7-L10) which is used by the PDK and Ruby, to run testing tasks.  This change called the helper script I created previously

* I modified [the AppVeyor configuration file](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/1039ebf83c4575c419aa4b70178b8afdad0da2a3/appveyor.yml) to call the new [`spec_pester`](https://github.com/puppetlabs/puppetlabs-wsus_client/blob/1039ebf83c4575c419aa4b70178b8afdad0da2a3/appveyor.yml#L33-L35) Rake task

Now whenever anyone raised a Pull Request, the pester test suite would be run!

Note - Why did I create a Rake task instead of calling the helper script directly? All the of PuppetLabs modules execute Rake tasks in the AppVeyor configuration file.  While I could have hacked the configuration to run the script directly it would cause this module to become a unique configuration which is hard to manage over time.

## Wrapping up

We modified the Bolt Task PowerShell script to be easily testable and then wrote a test suite.  We then configured our CI tool, AppVeyor, to run the tests for new Pull Requests.  Now we can more easily make changes to the task and be confident we don't break the existing behaviour

In Part 3, we'll look at how tasks, Puppet Enterprise and PowerShell can integrate together.

---

Changes to Part 1 of the blog

In Part 2, we'll look at how to test our newly created PowerShell Task
