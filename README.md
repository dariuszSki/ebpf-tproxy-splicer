# Intro:

  This is a project to develop an ebpf program that utilizes 
  tc-bpf to redirect ingress ipv4 udp/tcp flows toward specific
  dynamically created sockets used by openziti edge-routers.
  Note: For this to work the ziti-router code had to be modified to not insert
        ip tables tproxy rules for the services defined and to instead call map_update/map_delete_elem for tproxy redirection.
        example edge code at https://github.com/r-caamano/edge/tree/full_test assumes map_update and map_delete binaries are in ziti-router search path   
        and ebpf program loaded vi tc per below.  In a later release I will be working on writing the MAP update/delete directly via GO rather than via   
        system calls to external binaries.  Also note this is ebpf tc based so interception only occurs for traffic ingressing on the interface that 
        the ebpf program is running.  So clients on lan interface would be redirected but local router traffic would not be.  Have not tested running 
        on loopback at this point though may in the future.  Would likely need to change the restictive FW funtion if run on loopack. 

  prereqs: Ubuntu 22.04 server (kernel 5.15 or higher)

           sudo apt update

           sudo apt upgrade

           sudo reboot

           sudo apt install -y gcc clang libc6-dev-i386 libbpfcc-dev libbpf-dev

  compile:

        clang -O2 -Wall -Wextra -target bpf -c -o tproxy_splicer.o tproxy_splicer.c
        clang -O2 -Wall -Wextra -o map_update map_update.c
        clang -O2 -Wall -Wextra -o map_delete_elem map_delete_elem.c 
  
  attach:
        
        sudo tc qdisc add dev <interface name>  clsact

        sudo tc filter add dev <interface name> ingress bpf da obj tproxy_splicer.o sec sk_tproxy_splice

  detach:

        sudo tc qdisc del dev <interface name>  clsact

  Example: Insert map entry to direct SIP traffic destined for 172.16.240.0/24

        Usage: ./map_update <ip dest address or prefix> <prefix length> <low_port> <high_port> <tproxy_port> <protocol id>
        sudo ./map_update 172.16.240.0 24 5060 5060 58997 17 
 
  Example: Monitor ebpf trace messages

           sudo cat /sys/kernel/debug/tracing/trace_pipe
           
           ziggy@ebpf-router:~$ sudo cat /sys/kernel/debug/tracing/trace_pipe
           <idle>-0       [001] d.s.. 19039.327596: bpf_trace_printk: udp_tproxy_mapping->5060 to 54802
           <idle>-0       [001] dNs.. 19043.309122: bpf_trace_printk: udp_tproxy_mapping->5060 to 54802
           <idle>-0       [001] d.s.. 19043.736322: bpf_trace_printk: udp_tproxy_mapping->5060 to 54802
           <idle>-0       [001] d.s.. 19058.701643: bpf_trace_printk: tcp_tproxy_mapping->39999 to 36921
           <idle>-0       [001] d.s.. 19058.702262: bpf_trace_printk: tcp_tproxy_mapping->39999 to 36921
            <...>-8526    [001] d.s.. 19058.702852: bpf_trace_printk: tcp_tproxy_mapping->39999 to 36921
           <idle>-0       [001] d.s.. 19058.884039: bpf_trace_printk: tcp_tproxy_mapping->39999 to 36921
 
  Example: Remove prevoius entry from map

        Usage: ./map_delete_elem <ip dest address or prefix> <prefix len> <low_port> <protocol id>
        sudo ./map_delete_elem 172.16.240.0 24 5060 17
  
  