---
layout: post
title: "System Monitoring and Detections Using 'osquery'"
date: 2020-04-27 12:00:00 +0500
categories: forensics detection-tools osquery
---

**'osquery'** is an open-source tool which can be used to audit an operating system and all its configurations as an SQL-based relational database. It does so by exposing the OS, and representing abstract concepts of the OS (eg. processes, open sockets, kernel modules, etc.) as a series of SQL tables.

It was developed by Facebook and was later open-sourced for the community to take part in its development. It's cross-platform and has support for major operating systems like Windows, macOS, and Linux. 

# Getting Started

Since it's exposing the OS as a high-performance SQL database, writing SQL queries can help you extract information by means of its simple API. For example, if I was to extract information about the running processes on my machine, I'd run the command:

    SELECT * FROM processes;

Though I can assure you, the output would be quite ugly here. Why? There's loads of processes running simaltaneously which won't be fruitful if you quickly list them without relevant filters. Luckily, the SQL format allows you to restrict the matching rows to a number. Let's use the LIMIT keyword to limit our matches to 5.

    SELECT * FROM processes LIMIT 5;

Still, the table appears to have several tables. I'll start by listing the schema of the table, then select a few fields which are important to my research. Here's how you can view the schema and select a few columns:

    .schema processes
    SELECT pid, name, cmdline FROM processes LIMIT 5;

Where do you run these commands? Neither did I show you any output here. Let's discuss that next.

Installing osquery (available here) can help you install the following components at the same time:

- osqueryi: Interactive shell to write your queries
- osqueryd: Daemon for scheduling queries and run in the background

The background daemon tasks registers as a service and can run scheduled queries without distraction. The logs generated from these queries are also stored for aggregation, normalization, storage, or analysis with a SIEM solution. You can ship those off to Splunk, ElasticSearch (via LogStash), or whatever solution you'd like.

**Disclaimer: For the sake of this article, I'll be covering osquery on a Windows machine. You're free to test the tool on your choice of operating system. Only the installation and the availability of system tables should be different - the rest should be the same.**

Head over to this _[link]{:tagret="_blank"}_ in order to download an MSI package for osquery. Otherwise, you can also use chocolatey to setup osquery on your machine using the following command:

    choco install osquery

Out of the box, osquery has default configurations and flags which are either enabled or disabled. You can view your config files in the installation directory at this path:

    C:\Program Files\osquery

However, if you don't wish to use the default _osquery.conf_, you can use the _--config-path_ flag to reference a custom configuration file. You can do this when you're starting your interactive shell for osquery. Open up a command prompt and let's fire up osquery:

    osqueryi

![osquery-startup](/assets/osquery-detection/1.png)

If you wish to view the flagss you can use with it, run the command:

    osqueryi -help

Let's head back to our terminal. You can start scripting your ad-hoc queries here. Let's run _.help_ to see what options are available to us.

![osquery-help](/assets/osquery-detection/2.png)

You can view your current configuration by running the _.summary_ command on the interactive terminal. It'll also show you where the current configuration files are along with the loaded settings for the terminal itself.

# Setting up the Configuration File

We can setup the configuration file for osquery to make it easy to run it with several enabled flags and commands. Otherwise, you'd have to manually add them to the console every now and then.

Head over to the path:

    C:\Program Files\osquery

The _osquery.conf_ file can be used to configure the following:

- List of options and settings used by the daemon and the interactive shell
- Scheduling queries
- Enabling packs, which include several queries grouped to serve a specific purpose

We'll get back to editing this file. 

Similarly, we have the _osquery.flags_ file which can have the flags you'd use on the command line. By default, there are no flags applied to your interactive shell or daemon. You can open your flags file and add some options in there. Here's a look at my flags file, in which I've added a few settings to enable verbose standard outputs, windows events, along with the ability to run unsafe queries. 

![osquery-flags](/assets/osquery-detection/3.png)

Let's head back to the configuration file. Firstly, you can add in options for osuqeryi and osqueryd to make use of. I'll leave this section be, since we've added most of our flags in the .flags file. 

![osquery-options](/assets/osquery-detection/Screenshot_1.png)

Next, we have the schedule section, where we can easily schedule our queries and execute them in the suggested time. These are handled by the daemon itself. After that, their results are appended to the file, _osuqery.results.log_, which is available in the following directory:

    C:\Program Files\osquery\log\osquery.results.log

Here's the format of a scheduled query:

- Mention the name of the query under the key 'schedule'
- Create three key pairs inside this new object:
  - 'query' holds the actual query you'd execute
  - 'interval' holds the time range in which you wish to execute these
  - 'description' holds the description of the query (optional field)

Here's a sample scheduled query:

![osquery-options](/assets/osquery-detection/Screenshot_3.png)

After that, there's a special segment for 'decorators'. Decorators can add or append additional information to the queries you execute or schedule. For example, 

![osquery-options](/assets/osquery-detection/Screenshot_4.png)

The results from these queries are going to be appended to every output of your scheduled query. This is quite useful and can help identify the system or avoid information that's repetitive but equally important.

Lastly, we have packs. Packs can allow you to run specific queries. Here's a list of the packs which are included in our default installation (though not all of them are applicable to our installation):

![osquery-options](/assets/osquery-detection/Screenshot_5.png)

Let's uncomment the packs we do have and see if we can get them to work. Here's a list of a few queries in the windows-hardening pack. You can see how specific the queries are - for example, detecting the change of UAC to be disabled. 

![osquery-options](/assets/osquery-detection/Screenshot_6.png)

That's it for the configuration folks. If you'd like to read more about the configuratio in osquery, you can use this link: _[osquery-configuration]{:tagret="_blank"}_

# Ad-Hoc Queries Using osqueryi

Scheduling is great as you can't always script your queries or repeat them every now and then. But, in cases, you'd like to do that, osqueryi can help you write quick queries to gather data. In this section, I'll show you how you can run a few queries to extract some information.

Let's start off with logged in users. Here's the query for that:

![osquery-options](/assets/osquery-detection/Screenshot_7.png)

In this output, there is one real user account logged into the machine, and it's from a known IP address. If it’s not, you should investigate where that login came from.

You can view a complete list of tables using the command:

    .tables

View the schema and you can dissect important fields as well. Here's how you can view unique remote addresses via '**process_open_sockets**' on your network:

![osquery-options](/assets/osquery-detection/Screenshot_9.png)

If you just wish to see where are your processes running from, you can run the following command:

    SELECT path FROM processes;

Is there a process that you don't recognize? Or perhaps a sketchy binary being executed on your system? You can include in the PID and see if there's a network communication from that process. That can be done by joining several tables together by matching fields.

You can also use the **LIKE** construct if you're unsure about the value of a particular column. It should get you all relevant rows in the results. Here's an example if you'd try to find Chrome in your program listings:

![osquery-options](/assets/osquery-detection/Screenshot_10.png)

Here's how you can get the process name, port, and PID, for processes listening on all interfaces:

![osquery-options](/assets/osquery-detection/Screenshot_11.png)

# Conclusion

There's loads of other examples and use-cases. Using osquery for forensics is a great option. You can even use Kolide, which is built upon osquery, and can be used to manage an array of computers. We'll pick this article up in our next discussion on detecting more sophisticated malware, backdoors, and shells using osquery. 

That's it for today folks. See you in the next article! 

[link]: https://osquery.io/downloads/official/4.3.0
[osquery-configuration]: https://osquery.readthedocs.io/en/1.4.5/deployment/configuration/