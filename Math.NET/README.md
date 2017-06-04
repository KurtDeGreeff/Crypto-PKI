# PowerShell for Math.NET Numerics

PowerShell can utilize the math libraries of [Math.NET Numerics](http://numerics.mathdotnet.com/) and the processor-optimized [Intel Math Kernel Library (MKL)](https://en.wikipedia.org/wiki/Math_Kernel_Library). To help coders get started, this article is a collection of PowerShell examples using Math.NET with Intel's MKL.

## Background
Math.NET ([www.MathDotNet.com](http://www.mathdotnet.com)) is a set of open source mathematical toolkits optimized for ease of use and performance. The Math.NET libraries are compatible with the Microsoft .NET Framework and the Mono project, which makes them accessible to PowerShell too. The toolkits are available for free under the MIT/X11 license, which is even more liberal than the GNU Public License (except for the Intel MKL, which has its own license, but it is still [free for your own use](http://numerics.mathdotnet.com/MKL.html#Licensing-Restrictions) when obtained with Math.NET).

One part of the Math.NET project is [Math.NET Numerics](http://numerics.mathdotnet.com), which is especially useful for math computations in science and engineering. It includes methods for statistics, probability, random numbers, linear algebra, interpolation, regression, optimization problems, and more. The project includes hardware-optimized native libraries to maximize performance on x64 or x86 processors, such as Intel's Math Kernel Library (MKL).

The majority of computers used by scientists and engineers worldwide run Microsoft Windows. PowerShell is built into Windows and is relatively easy to learn. Think of PowerShell as "simplified C#" for use in a command shell and in scripts. Just like C#, PowerShell can access the .NET Framework; in fact, PowerShell itself is a .NET Framework application.

## Why PowerShell?
For scientists and engineers, PowerShell is great for utilizing the Intel Math Kernel Library (MKL) through Math.NET Numerics; managing long-running, complex compute jobs and scheduled tasks; orchestrating large-scale parallel MATLAB workloads in Azure; managing Machine Learning experiments in Azure; managing tasks on remote Linux boxes using a PowerShell port of ssh; programmatically interacting with REST/SOAP/XML/JSON web applications; doing quick-and-dirty distributed computing with Workflows and Remoting; running processes concurrently with RunSpaces or background jobs; importing/exporting data with Excel spreadsheets; quickly prototyping new ideas which could then be later ported to C# because of the similarities between PowerShell and C#; playing nice with the built-in support for Ubuntu bash on Windows 10 and later; and using the Posh-Git PowerShell module with GitHub Desktop.

For data visualization, PowerShell and Math.NET can export data for display in MATLAB or Excel, or you could script it yourself using the built-in .NET classes for [visualization](http://blogs.technet.com/b/richard_macdonald/archive/2009/04/28/3231887.aspx).

As an uncompiled and dynamically-typed language, PowerShell will never come close to the performance of C/C++, but with Math.NET and the Intel MKL provider, this might be less of an issue. On the other hand, C/C++ will never match the convenience (and fun) of using PowerShell with Math.NET or Python with NumPy. Can PowerShell script code be compiled? Yes and no: in some cases, parts of the script are compiled automatically on-the-fly, but we don't have much control over the JIT-ing when it occurs.


