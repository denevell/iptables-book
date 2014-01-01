********
iptables
********

.. contents::


Introduction: Basic concepts, setup and a basic rule
====================================================

Concepts: Tables, hook Points and chains
----------------------------------------

iptables have 'tables' that group the types of rules you can perform, e.g. forwarding packets, rejecting packets. By default, you have ``nat``, ``filter`` and ``mangle``, which will be explained in due course.

There are 'hook points' that allow you to hook into the network routing. At these points, you can insert your firewall rules. By default, you have ``PREROUTING``, ``INPUT``, ``FORWARD``, ``POSTROUTING`` and ``OUTPUT``, depending on what table you use. The rules attached to each hook point are refered to as  'chains'.

An example rule
---------------

Let's look at the following rule that rejects TCP/IP packets on port 1234.

	iptables -t filter -A INPUT -p tcp --dport 1234 -j REJECT

#. The ``-t filter`` part says we're adding to the 'filter' table, the table that allows us to reject a packet.
#. The ``-A INPUT`` part says we're appending to the chain of commands on ``INPUT`` hook point. Obviously, appending to the ``OUTPUT`` chain would be useless if we want to reject a packet hitting our network.
#. The ``-p tcp --dport 1234`` part states this rule should only apply to tcp connections and only connections which have 1234 as their destination port. 
#. Finally, ``-j REJECT`` states the target action, reject the packet in this case.
   
.. sidebar:: Default filter table

	The default table is the 'filter' table, so you could leave out the ``-t filter`` part in the above example.

Network setup for example programs
----------------------------------

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
----------------------------------------

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
------------------------------

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

Applying our rule
-----------------

In the above telnet session, we saw we connected to the program -- i.e. iptables allowed us to connect -- by the line 'Connected to 192.168.1.6.'.

Before we apply the rule we defined above, let's list all the rules in iptables, by running the command ``iptables -t filter -L -v`` as root:

.. code:: shell

	$ iptables -t filter -L -v
	Chain INPUT (policy ACCEPT 2193 packets, 893K bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain OUTPUT (policy ACCEPT 2123 packets, 485K bytes)
	 pkts bytes target     prot opt in     out     source               destination 

We can see that for the chains INPUT, FORWARD and OUTPUT in the table filter there are no rules defined.

Now let's apply our rule by issuing this command (yes, it's just the same as the rule above) as root: ``iptables -t filter -A INPUT -p tcp --dport 1234 -j REJECT``.

There should be no output from the above command, but if you run the listing command again you should see our new command:

.. code:: shell

	$ iptables -t filter -L -v                                 
	Chain INPUT (policy ACCEPT 1 packets, 164 bytes)
	 pkts bytes target     prot opt in     out     source               destination         
	    0     0 REJECT     tcp  --  any    any     anywhere             anywhere             tcp dpt:1234 reject-with icmp-port-unreachable

	Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

	Chain OUTPUT (policy ACCEPT 1 packets, 52 bytes)
	 pkts bytes target     prot opt in     out     source               destination         

The new line is telling us: 

#. If the protcol is TCP/IP, 
#. from any network interface to any other network interface, 
#. from any IP address to any other IP address, 
#. and the destination port is 1234,
#. then reject the packet with 'icmp-port-unreachable', the default response with you specify the REJECT target.
   
Communication with our program, rule in place
---------------------------------------------

As you may expect, if we try to connect to our program now, we'll get a rejected response. 

Here's the telnet output:

.. code:: shell

	$ telnet 192.168.1.6 1234
	Trying 192.168.1.6...
	telnet: Unable to connect to remote host: Connection refused
	$

Success!

If we now flush to iptables rules with ``iptables -F`` and then verify the rule is gone with ``iftables -L -v``, and try to connect again we will see the iptables rule is no longer in place.