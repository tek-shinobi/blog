---
title: "Backend C# Add NLog Logger in .NET project"
date: 2018-09-07T09:57:49+03:00
draft: false 
categories: ["backend"]
tags: ["c#"]
---

Recently, I was working on an enterprise project using Windows Forms and needed a logger that was thread-safe, allowed both structured and text based logging and provided an options to integrate email sending as well. Two options came standard NLog or log4Net.

Had a brief look at both. log4Net looked like more of XML configuration. NLog looked like less so. Went with NLog.


## How to add NLog to your .NET solution:

Scenario is that we have multiple projects in our solution.
Here are the steps:

1. Install NLog.Config through Nuget package manager. This will automatically install other dependencies like NLog and NLog.Schema. Please do this in your StartUp project.
1. Step 1 will add a file called NLog.config (we don’t modify another NLog.xsd file). Our goal is to set a new log file each day with date in it’s name and email notifications in case of warnings and (fatal) errors.

```xml
<variable name="logFilePath" value="C:\LPLogs\LP-${shortdate}.log"/>

<targets>
    <target name="logfile"
              xsi:type="File"
              fileName="${logFilePath}"
              layout="${longdate}   LEVEL=${level:upperCase=true}: ${message}${newline} (${stacktrace}) ${exception:format=tostring}"
              keepFileOpen="true" />
    
</targets>
<rules>
    <!-- add your logging rules here -->
    <logger name="*" minlevel="Info" writeTo="logfile" />
</rules>
```
With `${shortdate}` in the file name, it will automatically make a new logfile everyday.

Then, in which ever file you want to do logging, do like this (in my example, I am showing in Program.cs file)

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using NLog;

namespace App
{
    class Program
    {
        private static Logger logger = LogManager.GetCurrentClassLogger();
        static void Main(string[] args)
        {
            logger.Info("Entry Main method.");

            try
            {
                throw new Exception("Main exception.");
            }
            catch (Exception ex)
            {
                logger.Error(ex, "Main exception occurred.");
            }

            logger.Warn("A warning!");
            logger.Fatal("A fatal error...");
        }
    }
}
```
Note: Only lines 6 and 12 are important. Rest are added for testing NLog. Once again, whichever file needs logging, add lines 6 and 12 in it.  After you run this app, it will generate a logfile in the directory configured in NLog.Config (like in C:\LPLogs) with the dummy log data we put from lines 15 and 27.

If you want to log in other projects within the solution, just reference NLog.dll directly in each of the projects and use it. But now you only need to configure NLog in your main application. The configuration will then be shared automatically across all projects that reference NLog.dll from the main application.