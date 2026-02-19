# DevOps Labs

thor@jumphost ~$ ssh tony@172.16.238.10
The authenticity of host '172.16.238.10 (172.16.238.10)' can't be established.
ED25519 key fingerprint is SHA256:+36p5f6lI/bXZpFTx+vOWNW0Vgk52DQkGGlkS2FmBvA.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '172.16.238.10' (ED25519) to the list of known hosts.
tony@172.16.238.10's password: 
[tony@stapp01 ~]$ sudo yum install -y iptables-services

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony: 
CentOS Stream 9 - BaseOS                                                                        39 kB/s | 7.0 kB     00:00    
CentOS Stream 9 - BaseOS                                                                       8.3 MB/s | 8.9 MB     00:01    
CentOS Stream 9 - AppStream                                                                     48 kB/s | 7.1 kB     00:00    
CentOS Stream 9 - AppStream                                                                    9.0 MB/s |  27 MB     00:02    
CentOS Stream 9 - Extras packages                                                               40 kB/s | 7.6 kB     00:00    
CentOS Stream 9 - Extras packages                                                              5.3 kB/s |  20 kB     00:03    
Docker CE Stable - x86_64                                                                       24 kB/s | 2.0 kB     00:00    
Docker CE Stable - x86_64                                                                      326 kB/s |  68 kB     00:00    
Extra Packages for Enterprise Linux 9 - x86_64                                                 132 kB/s |  31 kB     00:00    
Extra Packages for Enterprise Linux 9 - x86_64                                                  18 MB/s |  20 MB     00:01    
Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64                           4.8 kB/s | 993  B     00:00    
Extra Packages for Enterprise Linux 9 - Next - x86_64                                           82 kB/s |  21 kB     00:00    
Extra Packages for Enterprise Linux 9 - Next - x86_64                                          344 kB/s | 259 kB     00:00    
Dependencies resolved.
===============================================================================================================================
 Package                             Architecture             Version                             Repository              Size
===============================================================================================================================
Installing:
 iptables-services                   noarch                   1.8.10-11.1.el9                     epel                    17 k

Transaction Summary
===============================================================================================================================
Install  1 Package

Total download size: 17 k
Installed size: 27 k
Downloading Packages:
iptables-services-1.8.10-11.1.el9.noarch.rpm                                                    50 kB/s |  17 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------
Total                                                                                           31 kB/s |  17 kB     00:00     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                       1/1 
  Installing       : iptables-services-1.8.10-11.1.el9.noarch                                                              1/1 
  Running scriptlet: iptables-services-1.8.10-11.1.el9.noarch                                                              1/1 
  Verifying        : iptables-services-1.8.10-11.1.el9.noarch                                                              1/1 

Installed:
  iptables-services-1.8.10-11.1.el9.noarch                                                                                     

Complete!
[tony@stapp01 ~]$ sudo iptables -A INPUT -p tcp -s 172.16.238.14 --dport 8089 -j ACCEPT
[tony@stapp01 ~]$ sudo iptables -A INPUT -p tcp --dport 8089 -j DROP
[tony@stapp01 ~]$ sudo service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables: [  OK  ]
[tony@stapp01 ~]$ sudo systemctl enable iptables
Created symlink /etc/systemd/system/multi-user.target.wants/iptables.service → /usr/lib/systemd/system/iptables.service.
[tony@stapp01 ~]$ sudo systemctl start iptables
[tony@stapp01 ~]$ sudo iptables -L -n
Chain INPUT (policy ACCEPT)
target     prot opt source               destination         
ACCEPT     tcp  --  172.16.238.14        0.0.0.0/0            tcp dpt:8089
DROP       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8089

Chain FORWARD (policy ACCEPT)
target     prot opt source               destination         
DOCKER-USER  all  --  0.0.0.0/0            0.0.0.0/0           
DOCKER-ISOLATION-STAGE-1  all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            ctstate RELATED,ESTABLISHED
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0           

Chain OUTPUT (policy ACCEPT)
target     prot opt source               destination         

Chain DOCKER (1 references)
target     prot opt source               destination         

Chain DOCKER-ISOLATION-STAGE-1 (1 references)
target     prot opt source               destination         
DOCKER-ISOLATION-STAGE-2  all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-ISOLATION-STAGE-2 (1 references)
target     prot opt source               destination         
DROP       all  --  0.0.0.0/0            0.0.0.0/0           
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           

Chain DOCKER-USER (1 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
[tony@stapp01 ~]$ 


<img width="1035" height="704" alt="image" src="https://github.com/user-attachments/assets/a2661489-5dd4-4da8-aa63-b7a91a870a7b" />
<img width="1031" height="857" alt="image" src="https://github.com/user-attachments/assets/05d818cc-a947-428c-a5b4-ad63a2e57b21" />
<img width="1032" height="858" alt="image" src="https://github.com/user-attachments/assets/ff5eed8f-24e0-4e10-a521-cc45c3bd9a6c" />
<img width="1034" height="547" alt="image" src="https://github.com/user-attachments/assets/5022598a-119c-4365-82c5-07c82ced067c" />
<img width="1030" height="402" alt="image" src="https://github.com/user-attachments/assets/d141897c-5e48-4dd3-bb04-af66e5b642cb" />
<img width="1032" height="440" alt="image" src="https://github.com/user-attachments/assets/83b44b3b-9334-4767-b65a-f426c5030fb4" />
<img width="1041" height="269" alt="image" src="https://github.com/user-attachments/assets/334a20a6-1d65-49be-9e61-a10264f77c66" />
<img width="1033" height="858" alt="image" src="https://github.com/user-attachments/assets/26e8c9cd-8010-40f7-8e13-a0b4b1794e94" />



