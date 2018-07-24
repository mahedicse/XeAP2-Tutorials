# XeAP 2 Session 2: NRO Lab<br>Setting up an eduroam National RADIUS Server


The XeAP 2 project will be using radsecproxy as the National and Top RADIUS Servers within the XeAP2 eduroam environment. There are other alternatives - such as FreeRADIUS - that are able to perform as an NRS but radsecproxy is lightweight and only requires the user to worry about a single configuration file unlike FreeRADIUS.

  
As the XeAP 2 Virutal Machines are running Ubuntu 18.04 LTS, we will install radsecproxy v1.7.1 from source as the apt package manager only provides version 1.6.9.
 

## radsecproxy Installation

1.  Update and refresh the Ubuntu package repositories to ensure we receive the most up-to-date packages:

		$ sudo apt update   

2. Upgrade currently installed packages via apt package manager:

		$ sudo apt upgrade

3. Reboot the virtual machine if a new kernel was installed:

		$ sudo reboot

4. Install packages needed to compile radsecproxy from source:

		$ sudo apt install build-essential libssl-dev make nettle-dev curl
		
5. Change directory to ```/usr/local/src/``` and download the radsecproxy v1.7.1 release
		
		$ cd /usr/local/src/
		$ sudo curl -Lo radsecproxy-1.7.1.tar.gz \
		      https://github.com/radsecproxy/radsecproxy/releases/download/1.7.1/radsecproxy-1.7.1.tar.gz

6. Extract the radsecproxy package, remove the package and enter into the radsecproxy source directory
		
		$ sudo tar xvf radsecproxy-1.7.1.tar.gz
		$ sudo rm radsecproxy-1.7.1.tar.gz
		$ cd radsecproxy-1.7.1

7. Download the radsecproxy patch needed to successfully compile the package on Ubuntu 18.04 LTS and apply patch to problematic file

		$ sudo curl -fsL -o tests/t_fticks.patch \
		    "https://raw.githubusercontent.com/spgreen/eduroam-radsecproxy-docker/master/1.7.1/patch/tests/t_fticks.patch"
		$ sudo patch tests/t_fticks.c tests/t_fticks.patch

8. Configure, compile, check and install radsecproxy -v

		$ ./configure
		$ make
		$ make check
		$ sudo make install

9. radsecproxy is now installed. The executable can be found using the ```which``` command
	
		$ which radsecproxy
		
	Output:
	
		/usr/local/sbin/radsecproxy
		
10. Check the radsecproxy version once installation has been completed:

		$ radsecproxy -v 

	Output you should receive:
	```
	radsecproxy revision 1.7.1
	This binary was built with support for the following transports:
	  UDP
	  TCP
	  TLS
	  DTLS
	```
	
11. Create an empty configuration file at the default radsecproxy configuration file location 

		$ touch /usr/local/etc/radsecproxy.conf

12. Open ```/usr/local/etc/radsecproxy.conf``` in your favourite text editor (vim, nano, emacs, etc)

		$ sudo vim /usr/local/etc/radsecproxy.conf
		
    **You are now ready to create the configuration for your new National RADIUS Server!**


## radsecproxy Configuration


1. Within the editor, add port, logging and F-Ticks statistical configuration 

	- **Listening interface and port number:**

	```
	# Listen on UDP port 1812 for all interfaces
	ListenUDP *:1812
	```

	- **General Logging:**

	```
	# Creates linelogs
	LogLevel 3
	
	# Write logs to /var/log/radsecproxy.log file
	LogDestination file:///var/log/radsecproxy.log
	```
	
	- **Loop prevention:**

		Semi automatic prevention loop for RADIUS connections. Defining a client and server block with the same descriptive name will prevent radsecproxy from proxying request to the same server. This is to ensure that only roaming user requests are sent to the proxy. <br>
	 	
		**Important Note:** You may wish to set this as ```	LoopPrevention Off``` for the XeAP2 workshop as loopbacks will be used for debugging purposes. 

	```
	LoopPrevention On 
	```
	
	- **Remove VLAN attributes:**

		Removes VLAN attributes that can be accidentally sent to the FLR even though it should be filtered out by the Service Provider (SP) before reaching the NRS. Identity Providers (IdPs) should not send this out at all. If the block was not in use and there were existing VLAN attributes, then the user would experience degraded or non-existent service.
	
    ```
    rewrite defaultclient {
        # IETF 64 (Tunnel Type)
        removeAttribute 64
        # IETF 65 (Tunnel Medium Type)
        removeAttribute 65
        # IETF 81 (Tunnel Private Group ID)
        removeAttribute 81
     }
     ```

2. Add Institutional RADIUS Servers (IRS) using Client, Server and Realm blocks. An IRS can act as an IdP, SP, or IdP & SP.
	
	- **Client Blocks**
	
		Client blocks are needed for eduroam Service Providers. You must communicate with the SP in regards to a secret so that the SP RADIUS server can communicate with the National RADIUS Server.?

    ```
    client IHL-SP_IdP-1 {
        # IP address or Fully Qualified Domain Name (FQDN) of the SP - example used
        host 203.0.113.1
        # RADIUS communication protocol used
        type UDP
        # Secret key negotiated between NRO and Institution
        secret changeme
    }
    ```

	- **Server Blocks**
	
		Server blocks are used for forwarding Access-Requests to an eduroam Identity Provider. Since the majority of IRS' will be both an IdP and a SP together, you will need to create a client and server block for each unique IRS.
    
    ```
    server IHL-SP_IdP-1 {
        host 203.0.113.1
        type UDP
        secret changeme
        # Periodically sends status checks to the IRS server
        statusserver on
   }
   ```

	- **Realm Blocks**
	
		Realm blocks indicate which IRS a roaming user's authentication request needs to be sent to. radsecproxy analyses the roaming user's realm (e.g. ```@ihl1-domain.edu.sg```)  from an authentication request, matches it with a Realm block and then sends it to the IRS belonging to said block. 
		
		Realm blocks use regular expressions to match patterns found within a realm. 
		For the realm block example below, it matches  ```@ihl1-domain.edu.sg``` and all sub-domains of ```ihl1-domain.edu.sg``` by using the following regular expression before the base domain: ```/(@|\.)```.
	
	```
	realm /(@|\.)ihl1-domain.edu.sg {
            server IHL-SP_IdP-1
	}
	```
	**Block Requirements**
		
	- **IRS as an Identity and Service Provider** - Client, Server and Realm blocks are required
	- **IRS as an Identity Provider** - Server and Realm blocks are required
	- **IRS as a Service Provider** - Only Client blocks are required


3. Add the blacklist filters

	- Uses regular expressions to check realm and since no server is specified, it will send back an Access-Reject packet with a replymessage to notify the Service Provider on why the Access-Request was rejected. This helps prevent bogus Access-Requests from being forwarded to the eduroam TLR.
 
	```
	realm /\s/ {
      replymessage "Misconfigured client: no route to white-space realm! Rejected by NRS."
	}

	realm /^[^@]+$/ {
      replymessage "Misconfigured client: no AT symbol in realm! Rejected by NRS."
	}

	realm /myabc\.com$ {
      replymessage "Misconfigured supplicant: default realm of Intel PRO/Wireless supplicant!"
	}

	realm /^$/ {
      replymessage "Misconfigured client: empty realm! Rejected by <TLD>."
      accountingresponse on
	}
	
	realm /(@|\.)outlook.com {
      replymessage "Misconfigured client: invalid eduroam realm."
      accountingresponse on
	}

	realm /(@|\.)live.com {
      replymessage "Misconfigured client:  invalid eduroam realm."
      accountingresponse on
	}

	realm /(@|\.)gmail.com {
      replymessage "Misconfigured client: invalid eduroam realm."
      accountingresponse on
	}

	realm /(@|\.)yahoo.c(n|om) {
      replymessage "Misconfigured client: invalid eduroam realm."
      accountingresponse on
	}
	```
4. Add the Top Level RADIUS (TLR) Client and Server blocks; Realm block will be added at the end.
	
	```
	# eduroam Top Level RADIUS blocks 
	
	client eduroam_TLR_1 {
	    type UDP
	    # Example IP address - Needs to be changed
	    host 198.51.100.1
	    secret __eduroam_secret-CHANGE-ME__
	}
	server eduroam_TLR_1 {
	    type UDP
	    # Example IP addres - Needs to be changed
	    host 198.51.100.1	
	    secret __eduroam_secret-CHANGE-ME__	
	    statusserver on
	}

	client eduroam_TLR_2 {
	    type UDP
	    # Example IP address - Needs to be changed	    
	    host 198.51.100.2	
	    secret __eduroam_secret-CHANGE-ME__
	} 
	server eduroam_TLR_2 {
	    type UDP
	    # Example IP address - Needs to be changed
	    host 198.51.100.2
	    secret __eduroam_secret-CHANGE-ME__
	    statusserver on
	}
	```
5. Add the eduroam Top Level RADIUS (TLR) Realm Block

	- The eduroam TLR Realm block forwards all other authentication requests to the TLR to perform additional routing if it is unable to find an appropriate IRS to authenticate the roaming user.

	```
	# DEFAULT forwarding: to the Top-Level Servers
	
	realm * {
	    server eduroam_TLR_1
	    server eduroam_TLR_2
	}
	```

**Example /etc/radsecproxy.conf**

```
#######################################################
#          GENERAL AND LOGGING CONFIGURATION          #
#######################################################
ListenUDP *:1812
LogLevel 3
LogDestination file:///var/log/radsecproxy.log

LoopPrevention On 

# Remove VLAN attributes
rewrite defaultclient {
    removeAttribute    64
    removeAttribute    65
    removeAttribute    81
}


#######################################################
#     eduroam IRS CLIENT, SERVER and REALM BLOCKS     #
#######################################################

# IHL-1 IdP and SP Block
client IHL-1-SP_IdP {
    host              203.0.113.1
    type              UDP
    secret            changeme
}	
server IHL-1-SP_IdP {
    host              203.0.113.1
    type              UDP
    secret            changeme
    statusserver      on
}
# IHL-1 Realm
realm /(@|\.)ihl1-domain.edu.sg {
	server	IHL-1-SP_IdP
}


#######################################################
#              BLACKLIST REALM FILTERS                #
#######################################################
realm /\s/ {
    replymessage "Misconfigured client: no route to white-space realm! Rejected by NRS."
}

realm /^[^@]+$/ {
    replymessage "Misconfigured client: no AT symbol in realm! Rejected by NRS."
}

realm /myabc\.com$ {
    replymessage "Misconfigured supplicant: default realm of Intel PRO/Wireless supplicant!"
}

realm /^$/ {
    replymessage "Misconfigured client: empty realm! Rejected by <TLD>."
    accountingresponse on
}

realm /(@|\.)outlook.com {
    replymessage "Misconfigured client: invalid eduroam realm."
    accountingresponse on
}

realm /(@|\.)live.com {
    replymessage "Misconfigured client:  invalid eduroam realm."
    accountingresponse on
}

realm /(@|\.)gmail.com {
    replymessage "Misconfigured client: invalid eduroam realm."
    accountingresponse on
}

realm /(@|\.)yahoo.c(n|om) {
    replymessage "Misconfigured client: invalid eduroam realm."
    accountingresponse on
}


#######################################################
#        eduroam TLR CLIENT and SERVER BLOCKS         #
####################################################### 

client eduroam_TLR_1 {
        type UDP
        host 198.51.100.1
        secret __eduroam_secret__
} 
server eduroam_TLR_1 {
        type UDP
        host 198.51.100.1
        secret __eduroam_secret__
        statusserver on
}

client eduroam_TLR_2 {
        type UDP
        host 198.51.100.2
        secret __eduroam_secret__
} 
server eduroam_TLR_2 {
        type UDP
        host 198.51.100.2
        secret __eduroam_secret__
        statusserver on
}

#######################################################
#                  TLR REALM BLOCK                    #
#######################################################

# DEFAULT forwarding: to the Top-Level Servers
realm * {
    server eduroam_TLR_1
    server eduroam_TLR_2
}
```

## Starting the National RADIUS Server 

There are two settings that are used to run the radsecproxy service. The first is in 'foreground' mode which we will use for debugging purposes. It will display logs to stdout as shown below.

1. Verify the configuration file is syntactically fine by running radsecroxy in pretend mode. There will only be output if there is an issue with the configuration file.

	```
	$ sudo radsecproxy -p	
	```
	
	
2. Run radsecproxy in foreground mode. Logs will be sent to stdout instead of the log file stated in ```/etc/radsecproxy.conf```. 

	```
	$ radsecproxy -f
	```
	```
	Jul 17 00:00:04 2018: createlistener: listening for udp on *:1812

	```


3. Stop radsecproxy by using ```Ctrl + c``` and start radsecproxy in the background.

	```
	$ sudo radsecproxy
	```

4. Check the log file to ensure radsecproxy logs are being received.

	```
	$ cat /var/log/radsecproxy.log
	```
	```
	Jul 11 00:00:04 2018: createlistener: listening for udp on *:1812

	```
  **Your NRS is now ready for production use.**

## Conclusion

The NRS is now setup and ready to accept RADIUS requests from trusted Institutional RADIUS Servers and the eduroam Top Level RADIUS. When new IRS joins you federation, be sure to add the appropriate Client, Server and/or Realm blocks depending on what type of service the IRS provides. Always ensure that the Top Level RADIUS  (TLR) Realm block is at the bottom of the configuration file. This will ensure that radsecproxy forwards only valid user requests to the eduroam TLR.


### Extra Configuration Information
For more in-depth explanations and configuration options, please visit the radsecproxy manual page located at [https://radsecproxy.github.io/radsecproxy.conf.html](https://radsecproxy.github.io/radsecproxy.conf.html "Offical radsecproxy.conf manual page").
