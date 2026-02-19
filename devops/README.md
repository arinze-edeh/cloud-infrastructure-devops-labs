# DevOps Labs

thor@jumphost ~$ ssh tony@stapp01
The authenticity of host 'stapp01 (172.16.238.10)' can't be established.
ED25519 key fingerprint is SHA256:4TjtVo6iuCgF6kRqHS+a8gUq3y8wKRGYoFp34Lv2z8k.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp01' (ED25519) to the list of known hosts.
tony@stapp01's password: 
[tony@stapp01 ~]$ sudo -i

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for tony: 
[root@stapp01 ~]# yum install iptables-services -y
CentOS Stream 9 - BaseOS                                                                        27 kB/s | 7.0 kB     00:00    
CentOS Stream 9 - BaseOS                                                                        14 MB/s | 8.9 MB     00:00    
CentOS Stream 9 - AppStream                                                                     30 kB/s | 7.1 kB     00:00    
CentOS Stream 9 - AppStream                                                                    3.7 MB/s |  27 MB     00:07    
CentOS Stream 9 - Extras packages                                                               54 kB/s | 7.6 kB     00:00    
CentOS Stream 9 - Extras packages                                                               49 kB/s |  20 kB     00:00    
Docker CE Stable - x86_64                                                                       30 kB/s | 2.0 kB     00:00    
Docker CE Stable - x86_64                                                                      377 kB/s |  68 kB     00:00    
Extra Packages for Enterprise Linux 9 - x86_64                                                 145 kB/s |  31 kB     00:00    
Extra Packages for Enterprise Linux 9 - x86_64                                                 1.5 MB/s |  20 MB     00:13    
Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64                           5.6 kB/s | 993  B     00:00    
Extra Packages for Enterprise Linux 9 - Next - x86_64                                           85 kB/s |  21 kB     00:00    
Extra Packages for Enterprise Linux 9 - Next - x86_64                                          438 kB/s | 259 kB     00:00    
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
iptables-services-1.8.10-11.1.el9.noarch.rpm                                                    66 kB/s |  17 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------
Total                                                                                           37 kB/s |  17 kB     00:00     
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
[root@stapp01 ~]# systemctl enable iptables
Created symlink /etc/systemd/system/multi-user.target.wants/iptables.service → /usr/lib/systemd/system/iptables.service.
[root@stapp01 ~]# systemctl start iptables
[root@stapp01 ~]# iptables -F
[root@stapp01 ~]# iptables -A INPUT -p tcp --dport 22 -j ACCEPT
[root@stapp01 ~]# iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT
[root@stapp01 ~]# iptables -A INPUT -p tcp --dport 5001 -j DROP
[root@stapp01 ~]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables: [  OK  ]
[root@stapp01 ~]# exit
logout
[tony@stapp01 ~]$ exit
logout
Connection to stapp01 closed.
thor@jumphost ~$ ssh banner@stapp03
The authenticity of host 'stapp03 (172.16.238.12)' can't be established.
ED25519 key fingerprint is SHA256:d5vNHcixiiXc9gIaEOpIcGZu8Aq+0Sn1q6yDzI1klwc.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp03' (ED25519) to the list of known hosts.
banner@stapp03's password: 
[banner@stapp03 ~]$ sudo -i

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for banner: 
[root@stapp03 ~]# yum install iptables-services -y
Last metadata expiration check: 0:14:25 ago on Thu Feb 19 05:05:58 2026.
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
iptables-services-1.8.10-11.1.el9.noarch.rpm                                                    71 kB/s |  17 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------
Total                                                                                           38 kB/s |  17 kB     00:00     
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
[root@stapp03 ~]# systemctl enable iptables
Created symlink /etc/systemd/system/multi-user.target.wants/iptables.service → /usr/lib/systemd/system/iptables.service.
[root@stapp03 ~]# systemctl start iptables
[root@stapp03 ~]# iptables -F
[root@stapp03 ~]# iptables -A INPUT -p tcp --dport 22 -j ACCEPT
[root@stapp03 ~]# iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT
[root@stapp03 ~]# iptables -A INPUT -p tcp --dport 5001 -j DROP
[root@stapp03 ~]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables: [  OK  ]
[root@stapp03 ~]# exit
logout
[banner@stapp03 ~]$ exit
logout
Connection to stapp03 closed.
thor@jumphost ~$ ssh steve@stapp02
The authenticity of host 'stapp02 (172.16.238.11)' can't be established.
ED25519 key fingerprint is SHA256:BkAPLco9Bjb0bn5g0HRp+XXoRBf95xoEpl1qJTQl3cw.
This key is not known by any other names
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'stapp02' (ED25519) to the list of known hosts.
steve@stapp02's password: 
[steve@stapp02 ~]$ sudo -i

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for steve: 
[root@stapp02 ~]# yum install iptables-services -y
CentOS Stream 9 - BaseOS                                                                        33 kB/s | 7.0 kB     00:00    
CentOS Stream 9 - BaseOS                                                                       1.6 MB/s | 8.9 MB     00:05    
CentOS Stream 9 - AppStream                                                                     45 kB/s | 7.1 kB     00:00    
CentOS Stream 9 - AppStream                                                                     29 MB/s |  27 MB     00:00    
CentOS Stream 9 - Extras packages                                                               32 kB/s | 7.6 kB     00:00    
CentOS Stream 9 - Extras packages                                                               49 kB/s |  20 kB     00:00    
Docker CE Stable - x86_64                                                                       30 kB/s | 2.0 kB     00:00    
Docker CE Stable - x86_64                                                                       11 kB/s |  68 kB     00:06    
Extra Packages for Enterprise Linux 9 - x86_64                                                 138 kB/s |  31 kB     00:00    
Extra Packages for Enterprise Linux 9 - x86_64                                                  23 MB/s |  20 MB     00:00    
Extra Packages for Enterprise Linux 9 openh264 (From Cisco) - x86_64                           6.2 kB/s | 993  B     00:00    
Extra Packages for Enterprise Linux 9 - Next - x86_64                                          115 kB/s |  21 kB     00:00    
Extra Packages for Enterprise Linux 9 - Next - x86_64                                          426 kB/s | 259 kB     00:00    
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
iptables-services-1.8.10-11.1.el9.noarch.rpm                                                    62 kB/s |  17 kB     00:00    
-------------------------------------------------------------------------------------------------------------------------------
Total                                                                                           29 kB/s |  17 kB     00:00     
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
[root@stapp02 ~]# systemctl enable iptables
Created symlink /etc/systemd/system/multi-user.target.wants/iptables.service → /usr/lib/systemd/system/iptables.service.
[root@stapp02 ~]# systemctl start iptables
[root@stapp02 ~]# iptables -F
[root@stapp02 ~]# iptables -A INPUT -p tcp --dport 22 -j ACCEPT
[root@stapp02 ~]# iptables -A INPUT -p tcp -s 172.16.238.14 --dport 5001 -j ACCEPT
[root@stapp02 ~]# iptables -A INPUT -p tcp --dport 5001 -j DROP
[root@stapp02 ~]# service iptables save
iptables: Saving firewall rules to /etc/sysconfig/iptables: [  OK  ]
[root@stapp02 ~]# exit
logout
[steve@stapp02 ~]$ exit
logout
Connection to stapp02 closed.
thor@jumphost ~$ 


<img width="1027" height="622" alt="image" src="https://github.com/user-attachments/assets/03f9b388-43df-402a-9580-07885d0881a5" />
<img width="1028" height="858" alt="image" src="https://github.com/user-attachments/assets/1783afb4-a794-414a-abef-b11cdaf60031" />
<img width="1033" height="605" alt="image" src="https://github.com/user-attachments/assets/db0dfae2-89e3-4e7a-8f03-327cbf368500" />
<img width="1032" height="486" alt="image" src="https://github.com/user-attachments/assets/ac087aa2-9658-4ac7-be94-6a231beea803" />
<img width="1036" height="565" alt="image" src="https://github.com/user-attachments/assets/3afd54bd-5748-4f68-94b4-3a47043e3699" />
<img width="1029" height="597" alt="image" src="https://github.com/user-attachments/assets/5cd488e0-80e3-488c-99c1-1eca6b816fff" />
<img width="1037" height="716" alt="image" src="https://github.com/user-attachments/assets/24a4c40e-b136-4daf-b774-08ed65d88926" />
<img width="1033" height="862" alt="image" src="https://github.com/user-attachments/assets/4d44f5d3-1b8c-4298-8f90-a1d4a42e0d77" />
<img width="1027" height="513" alt="image" src="https://github.com/user-attachments/assets/393378f3-bbaf-4723-ac85-c0d8af512ceb" />
<img width="1036" height="566" alt="image" src="https://github.com/user-attachments/assets/6a0111c8-2eb6-48db-9746-935e879fa216" />
<img width="1038" height="607" alt="image" src="https://github.com/user-attachments/assets/3959907a-ca46-46fd-9254-8bc0f80d9337" />
<img width="1034" height="465" alt="image" src="https://github.com/user-attachments/assets/6acd2809-2fe9-40ed-9058-69ef897b410e" />
<img width="1029" height="659" alt="image" src="https://github.com/user-attachments/assets/6a961336-8aa9-41e0-8696-877be33ebc00" />
<img width="1026" height="863" alt="image" src="https://github.com/user-attachments/assets/a2f9a6e3-e601-49f2-b192-05932ffed0b9" />
<img width="1033" height="616" alt="image" src="https://github.com/user-attachments/assets/68f72d67-ffa9-431e-a277-01ec23d3d0b8" />
<img width="1031" height="483" alt="image" src="https://github.com/user-attachments/assets/20d4d3f8-d112-4967-86ef-bda2e477b153" />
<img width="1030" height="638" alt="image" src="https://github.com/user-attachments/assets/c716f970-9789-42b6-a713-ddaff9159bb8" />
<img width="1033" height="600" alt="image" src="https://github.com/user-attachments/assets/d90cfa3a-ce76-4f8c-8788-83340b69f0b5" />
<img width="1034" height="690" alt="image" src="https://github.com/user-attachments/assets/cffa57ce-341d-4cc1-9546-8c8d8e797150" />


