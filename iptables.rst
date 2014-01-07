********
iptables
********

.. contents::

Bootstrapping your iptables knowledge
=====================================

This section will introduce you to the basics of iptables. We will do some setup for the examples, apply an iptables rule that rejects connections, apply a rule that accepts certain types of connections, change a default iptables policy, add some logging to it and along the way we'll look at the extensions to track connection state and limit matches.

Basic concepts, setup and a basic rule
--------------------------------------

Concepts: Tables, hook points and chains
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

iptables have 'tables' that group the types of rules you can perform, e.g. forwarding packets, rejecting packets. By default, you have ``nat``, ``filter`` and ``mangle``, which will be explained in due course.

There are 'hook points' that allow you to hook into the network routing. At these points, you can insert your firewall rules. By default, you have ``PREROUTING``, ``INPUT``, ``FORWARD``, ``POSTROUTING`` and ``OUTPUT``, depending on what table you use. The rules attached to each hook point are refered to as  'chains'.

An example rule
^^^^^^^^^^^^^^^

Let's look at the following rule that rejects TCP/IP packets on port 1234.

	``iptables -t filter -A INPUT -p tcp --dport 1234 -j REJECT``

#. The ``-t filter`` part says we're adding to the 'filter' table, the table that allows us to reject a packet.
#. The ``-A INPUT`` part says we're appending to the chain of commands on ``INPUT`` hook point. Obviously, appending to the ``OUTPUT`` chain would be useless if we want to reject a packet hitting our network.
#. The ``-p tcp --dport 1234`` part states this rule should only apply to tcp connections and only connections which have 1234 as their destination port. 
#. Finally, ``-j REJECT`` states the target action, reject the packet in this case.
   
.. sidebar:: Default filter table

	The default table is the 'filter' table, so you could leave out the ``-t filter`` part in the above example.

Network setup for example programs
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Normally, iptables controls traffic *between* networks, i.e your network and an external network. You may not, however, have the luxury of multiple networks to play with. As such, we will create a new virtual network. 

This will have the IP address of 192.168.1.6 on the virtual network device named 'tap0', for use in the forthcoming examples.

We are doing this since otherwise both our network program and client program would run on the same network, localhost. If this were the case, iptables would oftimetimes not bother controlling traffic that's using the same IP address on the same network. 

- First use the tunctl command, ``tunctl -t tap0``, to create a new virutal network device:

- Now run ``ifconfig -a`` and see that we have the new network interface:

::

	tap0	Link encap:Ethernet  HWaddr ae:f6:74:49:fc:d8
			BROADCAST MULTICAST  MTU:1500  Metric:1
			RX packets:0 errors:0 dropped:0 overruns:0 frame:0
			TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
			collisions:0 txqueuelen:500
			RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)		

- Since it doesn't have an ip address, let's give it one with ``ifconfig tap0 192.168.1.6 netmask 255.255.255.0 up``

- Now it should have an ip address that we can see if we run ``ifconfig``:

::

	tap0    Link encap:Ethernet  HWaddr ae:f6:74:49:fc:d8  
			inet addr:192.168.1.6  Bcast:192.168.1.255  Mask:255.255.255.0
			inet6 addr: fe80::acf6:74ff:fe49:fcd8/64 Scope:Link
			UP BROADCAST MULTICAST  MTU:1500  Metric:1
			RX packets:0 errors:0 dropped:0 overruns:0 frame:0
			TX packets:0 errors:0 dropped:2 overruns:0 carrier:0
			collisions:0 txqueuelen:500 
			RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)		

Network program for the examples
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will use a simple 'golang' program for our forthcoming examples. It listens for network connections on our newly created IP address above and outputs its input to the standard output.

Take the code below and save it in a file named 'net.go'.

.. code:: golang

	package main

	import "net"
	import "fmt"
	import "bufio"

	// Listens on connections to 192.168.1.6 on port 1234
	func main() {
	        ln, err := net.Listen("tcp", "192.168.1.6:1234")
	        if err!=nil {
	                fmt.Println("error listening: ", err)
	        } else {
	                fmt.Println("Listening")
	        }
	        for {
	                conn, err := ln.Accept()
	                if err!=nil {
	                        fmt.Println("error accepting: ", err)
	                        continue
	                } else {
	                        fmt.Println("Accepting a new connection")
	                }
	                go handleConnection(conn)
	        }
	}

	// On receiving a connection, just print out what was sent to it
	func handleConnection(conn net.Conn) {
	        bufferedReader := bufio.NewReader(conn)
	        for {
	                str, err := bufferedReader.ReadString('\n')
	                if err!=nil {
	                        fmt.Println("error reading: ", err)
	                        break;
	                } else {
	                        fmt.Print(str)
	                }
	        }
	}

We can start this by running ``go run net.go``.

Communicating with our program
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

We will use ``telnet`` to communicate with our example program. Here's an example of it in use:

.. code:: shell

	$ telnet 192.168.1.6 1234
	Trying 192.168.1.6...
	Connected to 192.168.1.6.
	Escape character is '^]'.
	This is an example.
	^]

	telnet> quit
	Connection closed.
	$

If we look at the output of our golang program we can see:

.. code:: shell

	$ go run net.go
	Listening
	Accepting a new connection

	This is an example.
	error reading:  EOF

.. sidebar:: error reading: EOF

	The 'error reading: EOF' came about when we pressed 'control ]' in telnet. It simply indicates the connection has been closed by the client sending an EOF to the program.

The program will continue to accept connections for its duration.

Listing and flushing our iptable rules
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Before we apply the rule we defined above, let's list all the rules in iptables, by running the command ``iptables -t filter -L -v`` as root:

.. code:: shell

	# iptables -t filter -L -v
	Chain INPUT (policy ACCEPT 2193 packets, 893K bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain OUTPUT (policy ACCEPT 2123 packets, 485K bytes)
	 pkts bytes target     prot opt in     out     source               destination 

We can see that for the chains INPUT, FORWARD and OUTPUT in the table filter there are no rules defined.

.. sidebar:: Default filter table

	Again, since the default table is 'filter', ``iptables -t filter -L -v`` is the same as ``iptables -L -v``

If we want to clear all the rules applied, we can flush them with 'iptables -F', which again works on the default 'filter' table unless specified otherwise.

Applying our rule
^^^^^^^^^^^^^^^^^

Now let's apply our rule by issuing this command as root. 

``iptables -t filter -A INPUT -i lo -d 192.168.1.6 -p tcp --dport 1234 -j REJECT``

We're slightly modifying what it was before to make it more precise. We're specifying an interface, ``lo``, and a destination, ``-d 192.168.1.6``. So the rule will match if the connection comes from localhost (our telnet command will send packets from this interface) and is directed at the specified IP address.

.. sidebar:: Precise iptables rules

	Above, we made the rule more precise. But if we'd left out 	``-d 192.168.1.6`` and ``-i lo``, we'd simply be saying match packets coming from any interface to any ip address, as long as they're going to a port number 1234.

There should be no output from the above command, but if you run the listing command again you should see our new command:

.. code:: shell

	# iptables -t filter -L -v                                 
	Chain INPUT (policy ACCEPT 1 packets, 164 bytes)
	 pkts bytes target     prot opt in     out     source               destination
	    0     0 REJECT     tcp  --  lo     any     anywhere             192.168.1.6          tcp dpt:1234 reject-with icmp-port-unreachable

	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain OUTPUT (policy ACCEPT 1 packets, 52 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

.. sidebar:: 'out' interface and source address

	Since we're blocking packets coming in on an interface, the 'out' interface option above is not applicable and therefore 'any'. And since we don't care where IP the packet comes from (nor can be reliably tell, oftentimes even on our machine), we're looking for source connections from 'anywhere'.

The new line is telling us: 

#. If the protcol is TCP/IP, 
#. from the 'lo', localhost, network interface
#. from any IP address to 192.168.1.6
#. and the destination port is 1234,
#. then reject the packet with 'icmp-port-unreachable', the default response with you specify the REJECT target.

Communication with our program, rule in place
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

As you may expect, if we try to connect to our program now, we'll get a rejected response. 

Here's the telnet output:

.. code:: shell

	$ telnet 192.168.1.6 1234
	Trying 192.168.1.6...
	telnet: Unable to connect to remote host: Connection refused
	$

Success!

If we now flush to iptables rules with ``iptables -F`` and then verify the rule is gone with ``iptables -L -v``, and try to connect again we will see the iptables rule is no longer in place.

Default chain policy
--------------------

If we look at the output when we list the iptables, we see a 'policy ACCEPT' line.

.. code:: shell

	# iptables -t filter -L -v
	Chain INPUT (policy ACCEPT 2193 packets, 893K bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain OUTPUT (policy ACCEPT 2123 packets, 485K bytes)
	 pkts bytes target     prot opt in     out     source               destination 

This means if no rules match, then we'll accept the packets.

Changing the policy to DROP
^^^^^^^^^^^^^^^^^^^^^^^^^^^

Let's change the default policy on ``INPUT`` to ``DROP`` via ``iptables -P INPUT DROP``.

.. code:: shell

	# iptables -P INPUT DROP
	# iptables -L -v
	Chain INPUT (policy DROP 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	
	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	
	Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination  

Now we can see the 'policy' line on ``INPUT`` now states ``DROP``, meaning we'd need to whitelist any open ports.

Whitelisting ESTABLISHED connections with the 'state' extension
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

This, however, means we will no longer be able to access the internet, since when we open a connection to a website, for example, it needs to send us back packets to display the webpage.

So now we need to allow iptables to accept connections when we open connections ourselves. This uses the 'state' match extension for iptables, enabled by default on my distributions.

	``iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT``

This line is similar to what we've seen before except the ``-m state`` means use an extension match module, 'state' in this case to match stateful connections. The ``--state`` line states that we're matching connections that have been established by us or related to a connection we started (which is the case in FTP).

Now we can talk to the internet just fine.

Logging
-------

Now our default chain policy on ``INPUT`` is DROP, we can put a logging action at the end of the chain. This will therefore catch any packets that are about to be dropped.

	``iptables -A INPUT -j LOG --log-prefix "dropped packet: "``

This inserts a rule at the end of our chain that simply logs the packet, outputting the phrase "dropped packet" to the beginnging of the line.

Listing our iptables filters should look like this now:

.. code:: shell

	# iptables -L -v
	Chain INPUT (policy DROP 1313 packets, 132K bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	16604   14M ACCEPT     all  --  any    any     anywhere             anywhere             state RELATED,ESTABLISHED
	 1313  132K LOG        all  --  any    any     anywhere             anywhere             LOG level warning prefix "dropped packet: "

.. sidebar:: Logging levels

	You can also use the ``--log-level`` parameter in the logging rule, which will pass a syslog logging level, with 7 being debug and 4 being warning, for example. By default, it is 'warning' as can be seen above.

You can find the logging in /var/log/syslog and via the command ``dmsg``.

Let's now try to connect to our program over telnet we can look at the logs via ``dmesg``.

::

	dropped packet: IN=lo OUT= MAC=... SRC=192.168.1.6 DST=192.168.1.6 LEN=60 ... PROTO=TCP SPT=46498 DPT=1234 ...
	dropped packet: IN=lo OUT= MAC=... SRC=192.168.1.6 DST=192.168.1.6 LEN=60 ... PROTO=TCP SPT=46498 DPT=1234 ...

We can now see iptables is rejecting any connection coming in to our program on 192.168.1.6:1234.

If we insert a rule to allow such connections, inserting it as the first rule with ``-I INPUT 1`` instead of ``-A INPUT``, we'll be allowed to access our program again and see no such logging rejecting packets to it.

.. code:: shell

	# iptables -I INPUT 1 -p tcp --dport 1234 -j ACCEPT                                                                                                                                         
	# iptables -L
	Chain INPUT (policy DROP)
	target     prot opt source               destination         
	ACCEPT     tcp  --  anywhere             anywhere             tcp dpt:1234
	ACCEPT     all  --  anywhere             anywhere             state RELATED,ESTABLISHED
	LOG        all  --  anywhere             anywhere             LOG level warning prefix "dropped packet: "
	...
	$ telnet 192.168.1.6 1234
	Trying 192.168.1.6...
	Connected to 192.168.1.6.
	Escape character is '^]'.

.. sidebar:: Log limiting

	If iptables is outputting a lot of logging -- which it may do in the cases of a netbios system constantly broadcasting packets to port 137 on the network, for example -- you may want to limit the amount of logging.
	
	The following command is as before, but it uses the 'limit' extension with the ``--limit 5/min`` to indicate that we should only log five entries, on average, every minute.
	
		``iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "dropped packet: "``
		
	You can also allow bursts of with the limit module ``--limit-burst 5``, which will allow five logs in one burst, and in the later minutes will try to keep to the average specified in the ``--limit directive``
	
	This limiting works for other types of actions, not just logging.

The filter table
================

Previously we dealt with accepting and rejecting packets on the INPUT chain of the filter table. We will now look at the OUTPUT and FORWARD chains of the filter table.

The OUTPUT chain
----------------

In the filter table, the OUTPUT chain will be hit when either a service, like our program above or an application like our web browser, tries to send a packet out from our network.

Let's use the ping program to talk to an external service via IP address. But to get that IP address, we'll use ``nslookup`` to find Wikipedia's IP address.

::

	$ nslookup wikipedia.com
	...
	Non-authoritative answer:
	Name:   wikipedia.com
	Address: 208.80.154.224

If we go to the IP address we found, it should go to a wikipedia page. More importantly, let's ping that address to make sure it's there.

::

	$ ping 208.80.154.224
	PING 208.80.154.224 (208.80.154.224) 56(84) bytes of data.
	64 bytes from 208.80.154.224: icmp_seq=1 ttl=54 time=195 ms
	64 bytes from 208.80.154.224: icmp_seq=2 ttl=54 time=117 ms
	64 bytes from 208.80.154.224: icmp_seq=3 ttl=54 time=117 ms
	
Now let's change our iptables OUTPUT chain to block access to that IP address.

::

	iptables -A OUTPUT -d 208.80.154.224 -j REJECT
	# iptables -L -v
	Chain INPUT (policy ACCEPT 114 packets, 23344 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	
	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	
	Chain OUTPUT (policy ACCEPT 99 packets, 9795 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	    4   336 REJECT     all  --  any    any     anywhere             text-lb.eqiad.wikimedia.org
	    
Now let's try that ping again. Press control-C quickly, else your terminal will fill up with 'Destination Port Unreachable' lines.

::

	$ ping 208.80.154.224
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable
	From 10.40.0.109 icmp_seq=1 Destination Port Unreachable

It's telling us that our local computer, 10.40.0.109 in my case, is telling the ``ping`` program that the destionation port is unreachable.

It's now a good idea to flush this rule, else we won't be able to access Wikipedia anymore.
