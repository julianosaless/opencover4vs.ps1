# opencover4vs.ps1
Smart powershell script for VisualStudio to generate code coverage report using OpenCore and ReportGenerator (and optionally publish them with Coveralls ). Configured with project files path globing (e.g *.Test.csproj). 

Works with Classic and Core frameworks (NUnit and xUnit Test projects). Windows only (not Linux).

HOW TO START
1. Install OpenCover and ReportGenerator as NUGET packages to one of your test projects (we need them to appear in packages folder).

2. Put script to the solution's root folder.

3. Configure those variables at the start
```
$TestProjectsGlobbing = @(,'*.Test.csproj')
$mstestPath = 'C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\Common7\IDE\MSTest.exe' 
$dotnetPath = 'C:\Program Files\dotnet\dotnet.exe'
$netcoreapp = 'netcoreapp1.1'
$NamespaceInclusiveFilters = @(,'*') # inlude all namespaces (which pdb found)
```

4. Run as PS script. You should find the report in the TestsResults\report\index.html 

There are some hidden assumptions about VS test projects:

   a) you can catch them and only them just using path globing, e.g. with pattern `'*.Test.csproj'` (but you can use several patterns)
   
   b) test projects are already COMPILED and theirs binaries are in project's output folder. 
   
   c) all xUnit projects from your solution use the same core framework (configured with `$netcoreapp` default 'netcoreapp1.1') and output folders are configured with the same path:
      
```   
   $classicProjectOutput = "bin\Debug"
   $coreProjectOutput = "bin\Debug\$netcoreapp"
```
   d) the test project's code (unit test classes) should use specific namespace (e.g. `MyApp.Test`) to build correct exclusive filter . If you want to escape building exclusive filters then configure `$BuildNamespaceExclusiveFilters = $false` (also do it for debugging the script). Important,  building exclusive filters for NUnit tests, this script collect namespaces of types that belongs to test assembly when for Core projects exclusive filter will be setuped with xUnit project's "Defaut Namespace".
         
   e) you should configure ALL of your Core projects to use full PDB format (Project Propertes>Build>Advanced) instead of default portable. Opencover doesn't understand portable PDB.

KNOWN PROBLEMS

NUnit attribute `Microsoft.VisualStudio.TestTools.UnitTesting.DeploymentItemAttribute` doesn't work properly. In scenarios like

    [TestClass]
    public class StandardConfigurationUnitTest
    {
        [TestMethod]
        [DeploymentItem("appsettings.json")]
        public void JsonConfigurationTest(){
                // ...    
        }
    }

this will not move `appsettings.json` to the `MSTest.exe` execution folder, therefore `appsettings.json` will be unaccessible for test code.

Note: In case of `/nointegration` option MSTest.exe working folder is the MSTest.exe location (like `C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\Common7\IDE\`); otherwise the working folder is MSTest.exe output folder e.g. `$SolutionFolder\TestResults\mstest\roman_DESKTOP-RLA3VAF 2017-04-04 04_22_07` but in any case those folders will not contain `appsettings.json` file so it can't be found. Changing OpenCover's working folder doesn't impact on MSTest working folder.


**Workouround**: recreate files from resources in unit test constructor.

    [TestClass]
    public class StandardConfigurationUnitTest
    {
        public StandardConfigurationUnitTest()
        {
            string fileName = "appsettings.json";
            using (var reader = new StreamReader(Assembly.GetExecutingAssembly()
                .GetManifestResourceStream(this.GetType().Namespace+"."+fileName)))
                using (var fileStream = new FileStream(fileName, FileMode.Create))
                    reader.BaseStream.CopyTo(fileStream);
        }
    }


**Other possible solution** is generate .testsettings file with "DeploymentItem" move action and point it as mstest configuration parameter.

```
	 <?xml version="1.0" encoding="UTF-8"?>
	 <TestSettings name="TestSettings1" id="1feaf251-3827-4529-8111-02e4aab1e4aa" xmlns="http://microsoft.com/schemas/VisualStudio/TeamTest/2010">
	 <Description>These are default test settings for a local test run.</Description>
	 <Deployment>
	   <DeploymentItem filename="Tests\Routines.Configuration.NETStandard.Test\bin\Debug\appsettings.json" />
	 </Deployment>
	 <Execution>
	 	<TestTypeSpecific>
	 		<UnitTestRunConfig testTypeId="13cdc9d9-ddb5-4fa4-a97d-d965ccfc6d4b">
         			<AssemblyResolution>
         					<TestDirectory useLoadContext="true" />
         			</AssemblyResolution>
	 		</UnitTestRunConfig>
	 	</TestTypeSpecific>
	  	<AgentRule name="LocalMachineDefaultRole"></AgentRule>
	  </Execution>
	  <Properties />
	</TestSettings>
```

   
