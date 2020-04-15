---
layout: post
title: "IBM QRadar: The Architecture!"
date: 2020-04-14 15:20:00 +0500
categories: siem ibm-qradar
---

Before you get started with the deployment of QRadar in your infrastructure, you need to understand the several components it makes use of to function properly. **IBM QRadar SIEM (Security Information and Event Management)** features a modular architecture where you can scale its deployment to add on more devices, endpoints, and machines in your infra to help with your analysis and logging needs. You can also add in modules to help with the analysis, which are easily provided by IBM on the App Exchange. The list includes but is not inclusive to:
- QRadar Vulnerability Manager
- QRadar Risk Manager
- QRadar Watson Integration
- QRadar Incident Forensics

# Three Layers: What Are They?

The QRadar architecture also follows the concept of layering and has three main layers through which the information flows from the endpoints to the agent. Mind you, this same architecture is followed in all sorts of deployments regardless of your organizations' size, complexity of your network, or the number of components utilized. 

But first...

## How Does it Work?

IBM QRadar collects, processes, aggregrates, correlates, stores, and presents the data from all subscribed endpoints in real-time. It'll use this cleaned data to provide you insight and information to help you monitor your IT Infrastructure. This is usually done in the form of alerts, offenses, and responses to threats which are caught red-handed.

You can further equipped some of the modules we discussed above to add more to your deployment. For example, the Incident Forensics module can help you break-down the the steps taken by a malicious actor and quickly draw conclusions based on the artifacts left behind. It can significantly reduce the time taken by an IR team to conduct the investigation on the offenses logged by QRadar itself. It'll also help you resolve breaches, prevent data losses, and attacks from ever happening again. 

_Back to the layers_ - let's discuss what those are and dive deeper into each one of them, next:

- Data collection
- Data processing
- Data searches

Take a look at the architecture yourself:

![IBM QRadar Architecture](/assets/qradar-architecture/arch.svg)

### Data Collection

'Data Collection' constitutes the bottom layer or the first layer of the architecture - and it's mission is simple "collect everything from your network". This includes data such as events, log files, flows, or any other information like scanned data, configuration files, packet captures, and so on. You can make use of the QRadar Event Collectors or QRadar QFlow Collectors (we'll discuss what 'flows' are again), to collect the event or flow data. 

QRadar can accept events from several log sources on your network. This can be your firewall or your Intrustion Prevention System (IPS), they're all valid log sources. To collect the data, it makes use of certain protocols, the most commonly used being syslog. Other options include: syslog-tcp, and SNMP. QRadar can also set up outbound connections to retrieve events by using protocols such as SCP, SFTP, FTP, JDBC, Check Point OPSEC, and SMB/CIFS.

Before it's readily available for your searches and analysis, the data goes through a series of transformations. The first is parsing. This is where the raw event data is transformed into a more readable and user-friendly format. That's done through the power of DSM Editor or the **Device Support Model** Editor. Through the DSM editor, you can quickly transform and parse your data into the format you'd like (collection of data via syslog, the protocol, is necessary before setting up a parser using the DSM editor).

After parsing, the normalization process takes over, which involves turning raw data into a format that has fields such as IP address that QRadar can use. Now, we talked about two main types of data here: _Event data and Flow data_. Let's discuss that:

- **Event Data:** Event data is comprised of activities or events which took place on a user's endpoint at a given point of time. This could be a firewall configuration change, a network failure, login failures, emails, connections that take place, or any other event that your device logs.

- **Flow Data:** Flow data is the information your networks generate upon communication or perhaps the session information. These events are transformed into "flow records". For example, this raw data (of two hosts communicating on a network) is transformed into an IP address, ports, bytes, packet counts, protocol or information that usually summarizes a sessions' information. One other thing, using the Incident Forensics module, there's also an ability to carry out a full packet capture besides gathering flow data. 

After normalization, QRadar recognized the event and the log source it's coming from. This is done by extracting relevant information from the IP header. These events are then coalesced into records. But, what if QRadar isn't able to detect the log source or it's unknown? The events are then sent off to the **traffic analysis engine** (used for auto-detection of the log sources). When new log sources are discovered, a configuration request message is sent off to the QRadar console to add that log source. 

In summary, the event collection layer has the following functions in order:

- Collect data using known protocol
- Monitor incoming events and manage them in queues for throttling
- Parse raw events into QRadar-usable fields
- Analyze the log source by means of DSMs which support automatic discovery
- Coalesce all events on common attributes
- Forward events to other SIEM solutions, targets, or systems

### Data Processing

After the data has completed its functions in the first layer, it's passed off to the second layer called the "processing" layer. Here, we have 'event processors' and 'flow processors' who're going to process the data that's collected and send it off to the final layer. What goes in here? Let's discuss.

The data is first run through the **Custom Rules Engine (CRE)** which is the first step. Here, the offenses and alerts are generated based on the data that is received and this data is written to storage for persistence. In the CRE, the custom rules, created by the users on the console (the main QRadar agent to be managed by analysts), are matched against the events. Each rule also has particular actions or responses which are set by the user itself. So, if the rule's conditions match the events, the actions will be taken. That's the job of the CRE.

The event processors are also responsible for streaming live data to the console. This is immediately available on the "Log Activity" section of the QRadar console. These events aren't replayed from the database but are operated on in a real-time fashion. However, they're also stored later for persistence. 

![Console](/assets/qradar-architecture/console.png)

There's one other level of abstraction we've left out before and that's **the Magistrate**. The CRE's don't pass out matched events directly to the console. They're sent to the Magistrate on the QRadar console, which creates and manages offenses on the console. It has three main tasks:

- Offense Rules: What to do when an offense has generated?
- Offense Management: Manage the offenses, update their status, provide more information, etc.
- Offense Storage: Events are later stored in a Postgres Database

Similarly, The **Magistrate Processing Core (MPC)** is responsible for correlating offenses with event notifications from multiple Event Processor components.

### Data Searches

Finally, the normalized and processed data is sent to the final layer on QRadar, which makes it available for searches by users. This can help you analyze, report, and investigate on alerts, rules, and offesnes on the QRadar console. The QRadar console also allows analysts to take actions and perform administration tasks as required. 

In distributed environments, the QRadar Console does not perform event and flow processing, or storage. Instead, the QRadar Console is used primarily as the user interface where users can use it for searches, reports, alerts, and investigations.


# Conclusion

That's it for today folks. I'll be concluding my article on the IBM QRadar architecture. We'll be back on the same topics again in a few days, till then - farewell! Stay safe!