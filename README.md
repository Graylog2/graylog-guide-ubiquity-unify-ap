# graylog-guide-ubiquity-unify-ap

This guide explains how to configure a [Ubiquity Networks Unifi Enterprise WiFi Access Point](https://www.ubnt.com/unifi/unifi-ap/) to send logs to Graylog and how to configure Graylog to parse these into nicely structured messages.

![](https://github.com/Graylog2/graylog-guide-ubiquity-unify-ap/blob/master/message.png)

## Configuring Graylog

1. Start a _Syslog UDP_ input and remember the port you let it listen on. You'll need it later when you are pointing your access points to Graylog.
1. Create a stream and call it _Ubiquity Access Point logs_
1. Add one stream rule: `message must match regular expression ^\(".+,(.+?),.+"\) (.+?): (.+)$`
1. Create a pipeline with one stage and two steps:
  * Parse the actual log message into fields and clean it up
  * Search for any mac address in the message and add it as another field

Here are the rules:

```
rule "parse Ubiquity access point logs"
when
  has_field("message")
then
  let m = regex("^\\(\".+,(.+?),.+\"\\) (.+?): (.+)$", to_string($message.message));
  
  let bssid = m["0"];
  let subsystem = m["1"];
  let clean_message = m["2"];
  
  // Build a better source name
  set_field("source", concat("ap-", to_string(bssid)));
  
  // Set additional fields.
  set_field("type", "ubiquity-ap");
  set_field("bssid", bssid);
  set_field("subsystem", subsystem); 

  // Set a better message field without the prefix clutter.
  set_field("message", clean_message);
end
```

```
rule "parse any MAC address out of message field"
when
  has_field("message")
then
  let m = regex("([0-9A-Fa-f]{2}:[0-9A-Fa-f]{2}:[0-9A-Fa-f]{2}:[0-9A-Fa-f]{2}:[0-9A-Fa-f]{2}:[0-9A-Fa-f]{2})", to_string($message.message));
  
  // It's NULL if there was no match and will simply not be set internally by Graylog.
  set_field("mac_address", m["0"]);
end
```

![](https://github.com/Graylog2/graylog-guide-ubiquity-unify-ap/blob/master/pipeline.png)

Connect this pipeline to your _Ubiquity Access Point logs_ stream and you are done on the Graylog side.

## Configuring the Access Point

In Graylog, start a 

In your Unifi Web Interface, go to "Settings" and enable remote syslog logging. Use the port that your _Syslog UDP_ input in Graylog is using:

![](https://github.com/Graylog2/graylog-guide-ubiquity-unify-ap/blob/master/unifi.jpg)
