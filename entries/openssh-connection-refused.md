# Troubleshooting OpenSSH
A simple search on the internet for issues related to SSH would probably solve most if not all of the problems. This document, however, covers scenarios where the problem is not directly related to OpenSSH or its configuration.

- [Troubleshooting OpenSSH](#troubleshooting-openssh)
  - [Connection refused](#connection-refused)
    - [Scenario 1: Connection refused when connecting to an Azure Virtual Machine provisioned using CLI](#scenario-1-connection-refused-when-connecting-to-an-azure-virtual-machine-provisioned-using-cli)

## Connection refused  
### Scenario 1: Connection refused when connecting to an Azure Virtual Machine provisioned using CLI

Identifed: Sun Jan 27 2 AM CET 2022  
Solved: Sun Jan 27 11 AM CET 2022  

**Premise.**  
I was working on a GitHub Actions workflow which would provision a VM and then use Ansible to customize the VM and finally create a Managed Image from the VM. The VM provisioning was done using the Azure CLI and ssh was used for authentication. This setup was working correctly untill it didnt. A complete run of this workflow would take 2-3 hours and when the SSH connection fails, its not easy to "resume" this run or to do a quick analysis (not at 3 am atleast).

    TASK [Gathering Facts] 
    *********************************************************
    fatal: [172.16.0.5]: UNREACHABLE! => {
        "changed": false,
       "unreachable": true
    }
    MSG:
    Failed to connect to the host via ssh: ssh: connect to host 172.16.0.5 port 22: Connection refused

    PLAY RECAP 
    *********************************************************************
    172.16.0.5                  : ok=0    changed=0    unreachable=1    failed=0    skipped=0    rescued=0    ignored=0   

    Error: Process completed with exit code 4.

Naturally, I checked if I could log into this VM using SSH, and I was able to, successfully. I even checked if the sshd.service was active and running (Yes, I was aware that I logged in using ssh). This is important to keep in mind throughout this scenario, **sshd.service was active and running on the server**.

Next, I checked if there was any login errors in the journal, and there were a few

    srini@vm $ journalctl -xe -u sshd  
    ...
    ...
    sshd[711706]: Connection closed by 172.16.0.4 port 52122 [preauth]  
    sshd[711706]: Connection closed by 172.16.0.4 port 52153 [preauth]  
    sshd[711706]: Connection closed by 172.16.0.4 port 52178 [preauth]  
    ...

"[preauth]" indicates that this is not an authentication related issue but I was trying to debug at 3 AM (bad idea), so I checked the user and keys anyway.

**First breakthrough.**  
There were no other error messages reported by the sshd service, so I tried searching online for a possible solution and I found all the usual suggestions, but to repeat again, I tried logging in from the VM which was opening the SSH connections, to the same user on the Azure VM using the same private key. Everything worked as expected. So I started another workflow while still trying to identity the problem. The workflow run failed (obviously) with the exact same error, but I added some extra options (**ssh -vvv** and **ansible-playbook -vvvvv**) to increase the output verbosity.  
**This last run confirmed that the issue was client side and not server side as I had initially thought.**

The output from the ssh connection attempt was very revealing as it was very empty in comparison to a successful connection attempt.

      Failed to connect to the host via ssh: OpenSSH_8.2p1 Ubuntu-4ubuntu0.4, OpenSSL 1.1.1f  31 Mar 2020  
      debug1: Reading configuration data /etc/ssh/ssh_config  
      debug1: /etc/ssh/ssh_config line 19: include /etc/ssh/ssh_config.d/*.conf matched no files  
      debug1: /etc/ssh/ssh_config line 21: Applying options for *  
      debug2: resolve_canonicalize: hostname 172.16.0.5 is address  
      debug1: auto-mux: Trying existing master  
      debug1: Control socket "/home/srini/.ansible/cp/32a3b79a72" does not exist  
      debug2: ssh_connect_direct  
      debug1: Connecting to 172.16.0.5 [172.16.0.5] port 22.  
      debug2: fd 3 setting O_NONBLOCK  
      debug1: connect to address 172.16.0.5 port 22: Connection refused  
      ssh: connect to host 172.16.0.5 port 22: Connection refused  

Normally there would be a lot more messages sent by the client/server  between following two lines

      debug1: connect to address 172.16.0.5 port 22: Connection refused  
      ssh: connect to host 172.16.0.5 port 22: Connection refused  

Since there were no messages sent after the TCP "connect to address 172.16.0.5", I made the following assumptions (at 3 AM)  
1. TCP connection was successful. I was able to connect when I tried manually
2. OpenSSH client was not sending any messages during the workflow. I assumed this might have something to do with the shell environment during the connection attempt.
3. OpenSSH server was immediately closing the connection request. This is just a different version of the previous assumption
4. The connection attempt from Ansible was failing, may be it was related to an Ansible module.

With these ideas in mind, I went to sleep. It was almost 4 and I wasnt thinking properly (see 4. above). I did not destroy the most recently created environment because I wanted to resume debugging immediately in the morning.

**The next morning.**  
I was determined to verify all 3 assumptions. The 4th assumption was not necessary, as any ssh connection attempts before the ansible-playbook command also failed.

*Assumption 1: TCP connection was successful*  
I tried to check if the port 22 was open on the VM (172.16.0.5) from the VM (172.16.0.4).

    srini@vm $ nc -vz 172.16.0.5 22  
    Ncat: Version 7.91 ( https://nmap.org/ncat )  
    Ncat: Connected to 172.16.0.5:22.  
    Ncat: 0 bytes sent, 0 bytes received in 0.06 seconds.  

**The two VMs are in the same subnet 172.16.0.0/24 and there were no firewalls or network security groups or such that could possibly drop the traffic. Therefore there is absolutely nothing wrong with the TCP connection.**

*Assumption 2: OpenSSH client was not sending any messages*  
Increasing the verbose level on the the client did not provide any more detail. So the next option was to inspect the packets themselves. So I opened up a terminal on the VM (172.16.0.4) and started listening for outgoing packets. I use tshark (Wireshark CLI) because I like the way filters can be written in tshark compared to tcpdump (which is built-in, mostly).

We cannot filter for packets by port, that would also show packets from my own session. So we listen packets based on src (source) and dst (destination) IP addresses on the interface 'ens3'.

    srini@vm $ tshark -i ens3 -f "src 172.16.0.5 or dst 172.16.0.5"  
    Capturing on 'ens3'  
      3 0.000000000 172.16.0.4 → 172.16.0.5  TCP 74 52024 → 22 [SYN] Seq=0 Win=29200 Len=0 MSS=1460 SACK_PERM=1 TSval=3732464338 TSecr=0 WS=128   
      4 0.000588317 172.16.0.5 → 172.16.0.4  TCP 54 22 → 52024 [RST, ACK] Seq=1 Ack=1 Win=0 Len=0 

I removed the extra packets related to ARP and the PDU (Protocol Data Unit) information from the above capture because the information needed to solve the problem is already displayed.

The '[RST, ACK]' in the last packet revealed the reason for the connection refused scenario. The L4PDU only confirms it  

        [SEQ/ACK analysis]
            [This is an ACK to the segment in frame: 1]
            [The RTT to ACK the segment was: 0.000088345 seconds]
            [iRTT: 0.000088345 seconds]
        [Timestamps]
            [Time since first frame in this TCP stream: 0.000088345 seconds]
            [Time since previous frame in this TCP stream: 0.000088345 seconds]

The TCP connection was closed almost immediately by the server and the client had no opportunity to continue with the SSH protocol, hence the lack of messages between connect and the connection refused error message.  

> 1. Assumption 1. was incorrect.  
> 2. TCP connection was not successful and it was a server side error after all.   
> 3. OpenSSH server was not ready and running when the SSH connection was attempted, it was however when I checked it manually.

**Solution**  
Once the problem was identified, it was easy to come up with a solution. All I had to do was wait untill the SSH service was running before proceeding with the remainder of the script.  

    # IP_ADDRESS=172.16.0.5
    # PORT=22
    set +e
    nc -w 1 -vz "$IP_ADDRESS" "$PORT"
    Status=$?
    while [ "$Status" -ne 0 ]
    do
      echo "Waiting for OpenSSH service"
      sleep 5
      nc -w 1 -vz "$IP_ADDRESS" "$PORT"
      Status=$?
    done
    echo "OpenSSH is working"
    set -e

The "set +e" is needed since "errexit" option is set by default and it will terminate the pipeline early if the nc command fails. And we renable the "errexit" option after the while loop (Using + rather than - causes these options to be turned off, see "man set" for more options).

Using a fixed timeout for nc is not recommended since this problem is uncommon and the ssh service might become ready very early or very late depending on various factors. Having a loop that checks if the ssh service is running adds a slight overhead but avoids the situation where a pipeline fail happens 

**tl;dr**  
1. I tried to ssh into a remote virtual machine immediately after the creation API request was completed.
2. sshd.service was not running when the initial connection was attempted but it was running properly seconds/minutes afterwards which led to me making an incorrect assumption about the problem.