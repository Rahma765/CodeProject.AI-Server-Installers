# Creating the Inno Setup

## Install Inno Setup
Inno Setup can be installed from https://jrsoftware.org/download.php/is.exe. Run the installer and it will be installed in *C:\Program Files (x86)\Inno Setup 6*,

## Install the Windows SDK signing tool

[Download Windows SDK](https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/) but only install the code signing bits. [Review this](https://www.digicert.com/kb/code-signing/ev-authenticode-certificates.htm) and then download the [Safenet drivers and client tool](https://knowledge.digicert.com/solution/how-to-install-the-safenet-drivers-and-client-software-windows)

## Building the Setup file
To create the setup file, run the *build_inno.bat* file. In Visual Studio, Mads' [Open Command Line](https://marketplace.visualstudio.com/items?itemName=MadsKristensen.OpenCommandLine64) extension makes this easy. 

The batch file will:

1. rebuild the solution
1. run the inno setup compiler
1. the output will be in the *Output* subdirectory in the file *CodeProject.AI-Server-win-x64-x.y.x.exe* file where x,y,z are the version parts.
1. The file needs to be signed, and a zip created, before deploying to the download host.

## Advanced Setup creation
If you want to modify and test the setup script and test it you can run the [ISS compiler IDE]("C:\Program Files (x86)\Inno Setup 6\Compil32.exe"). This allows you to edit, compile, and run the installer/uninstaller. It provide detailed output, including errors on compile and run. Intellisense is also provided when writing code.

The help is very usable, and the only really usable documentation I've found so far.