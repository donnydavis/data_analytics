In this dead simple blog we are going to show you how to use the EFK stack
to do logging from pfsense without complicated groks and zero regex.

I couldn't find a post on the web for parsing pfsense firewall logs,
so I just decieded to figure it out with the simplest possible method.

### Prerequisites
  - Functional EFK cluster - I am using centos opstools because it was super simple to get started
  - PFSense
  - Traffic to log

Now with that out of the way, we can get started.

This is real.... and I mean real easy. PFSense 2.2+ sends out firewall logs in csv format
https://doc.pfsense.org/index.php/Filter_Log_Format_for_pfSense_2.2

This is fantastic because fluentd has a built in mechanism for parsing csv data.
https://docs.fluentd.org/v0.12/articles/parser_csv

1. Setup PFSense to send logs to a remote location
Status>System Logs>Settings
Enable Send log messages to remote syslog server
Remote log servers $EFKIP:5142
Remote Syslog Contents [X] Everything

Now your pfsense box will send its logs to fluentd on for 5142

2. Setup EFK to listen and receive on 5142

We are going to tell fluentd to listen on port 5142 for syslog formatted messages.
We are also going to tag these messages with pfsense.messages. (pick whatever you want here)
```
<source>
  @type syslog
  port 5142
  bind $EFKIP
  tag pfsense.messages
</source>
```

Next we want to pull out just the filterlog from the syslog message and dump everything else (We will make use of the other logs in a different post)
This function rewrites the messages with the host field of filterlog and retags them to pf_log
```
<match pfsense.**>
  type rewrite_tag_filter
  rewriterule1 host ^filterlog  pf_logs # Get logs from filterlog
  rewriterule2 .*                   clear # all the other logs are sent to a different tag
</match>
```

Now that we have the data we are looking for, lets transform that data into something usable.
Because pfsense sends firewall logs in csv format, we are going to pull the message field and then parse it using the csv parser
The keys field is important here, because we are labeling what we want to see in kibana using these keys.
```
<match pf_logs>
  type parser
  key_name message
  format csv
  keys rule,sub_rule,anchor,tracker,interface,reason,action,direction,ip_ver,tos,ecn,ttl,id,offset,flags,protocol_id,protocol,length,source,destination,source_port,destination_port,data_length,tcp_flags,seq_number,ack,window,urg,options
  null_value_pattern '-'
  tag pf_parsed
</match>
```

So we have important data, and now its labeled how we want it to be. Next we are going to retag each type of traffic
I break apart each type of traffic into its own tag so I can manipulate it.
```
<match pf_parsed>
  type rewrite_tag_filter
  rewriterule1 protocol ^tcp  pf_tcp # rewrite tcp traffic
  rewriterule2 protocol ^udp  pf_udp # rewrite udp traffic
  rewriterule3 protocol ^icmp pf_icmp # rewrite icmp traffic
  rewriterule4 .*                   clear
</match>
```

We are getting close here. The next step is to take each tag and filter what we don't want. This is entirely subjective and not required.
```
You don't need this, but the values just come up nil in most cases
<filter pf_tcp>
  @type record_transformer
  enable_ruby true
  remove_keys rule,sub_rule,anchor,ecn
</filter>

This section pulls the extra fields from the transform. UDP does not have tcp_flags, seq_number,ack etc
<filter pf_udp>
  @type record_transformer
  enable_ruby true
  remove_keys rule,sub_rule,anchor,ecn,tcp_flags,seq_number,ack,window,urg,options
</filter>

This section does the same as above, but for the source_port we need to transform to icmp_type because that is how it's logged.
I left the options field available for extra data sent by pfsense
<filter pf_icmp>
  @type record_transformer
  enable_ruby true
  <record>
    icmp_type ${record["source_port"]}
  </record>
  remove_keys rule,sub_rule,anchor,ecn,source_port,destination_port,data_length,tcp_flags,seq_number,ack,window,urg
</filter>
```

Done. Now we have parsed and filtered logs. The final step is to now send them somewhere so they can be viewed
```
<match pf_tcp>
  @type elasticsearch
  logstash_format true
</match>
<match pf_udp>
  @type elasticsearch
  logstash_format true
</match>
<match pf_icmp>
  @type elasticsearch
  logstash_format true
</match>
```

And that is all folks, now you have data in EFK that you can sort through. Fluentd is a powerful tool for log analytics.

Please see 100-listen-pfsense.conf for a file that can be put into directly into Fluentd!

Thanks
~DonnyD
