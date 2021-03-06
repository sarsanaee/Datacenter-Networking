### DCB configuration

I trying to do the DCB configuration for `Ethernet controller: Intel Corporation 82599ES 10-Gigabit SFI/SFP+ Network Connection`, 
I know a little about DCB(Data Center Bridging) but it seems this DCB and VMDQ(Virtual Machine Device Queue) are all together usually.
 There is a DPDK sample application configures DCP + VMQD together. It is located at `DPDK_ROOT/example/vmdq`, there is also a VMDQ only
 example at `DPDK_ROOT/example/vmdq_dcb`. For now I only intend to configure two traffic classes with DCB.

A sequence of commands comes handy and I want to keep them here!

My first trial is with the DPDK testpmd application, here is how I start my testpmd application:
1. `sudo ./testpmd -l 0-8 -n 4 -- -i --portmask=0x1 --nb-cores=8`  
    1. Initally we need to start testpmd app and I just give it 8 cores(maybe it is 9 cores) and 4 memory channels, and then just a mask to
to select a port(I have two of them and just need one so 0x1 suffices). Also I need to determine how many cores each port requires,
and now it is 8.
2. `show port dcb_tc 0`
    1. This shows us the current status of the port! Regarding DCB configurations. For now I don't know what RXQ/TXQ base/number is.
    2. We suppose to get the following results:
    ```================ DCB infos for port 0   ================
       TC NUMBER: 1

       TC :        	   0
       Priority :  	   0
       BW percent :	  12%
       RXQ base :  	   0
       RXQ number :	   0
       TXQ base :  	   0
       TXQ number :	   0
    ```
    3. Obviously we have one traffic class out of 8 traffic classes possible on each port!
3. `port stop 0`
    1. To apply for configuration, we must stop the port first.
4. `port config 0 dcb vt on 8 pfc off`
     1. I don't want any PFC for now, I just want to configure the traffic classes.
     2. I should have 8 traffic classes if I get the status again:
     ```  ================ DCB infos for port 0   ================
     TC NUMBER: 8

     TC :        	   0	   1	   2	   3	   4	   5	   6	   7
     Priority :  	   0	   1	   2	   3	   4	   5	   6	   7
     BW percent :	  12%	  13%	  12%	  13%	  12%	  13%	  12%	  13%
     RXQ base :  	   0	   1	   2	   3	   4	   5	   6	   7
     RXQ number :	   1	   1	   1	   1	   1	   1	   1	   1
     TXQ base :  	   0	   1	   2	   3	   4	   5	   6	   7
     TXQ number :	   1	   1	   1	   1	   1	   1	   1	   1
     ```
5. `sudo ./testpmd -l 0-10 -n 4 -- -i --portmask=0x1 --nb-cores=10 --rxq=10 --txq=10`
     1. This is fair enough, I just tried some different command line options to investigate more.
     
### BESS DCB configuration
NICs are connected to BESS through DPDK so no Linux based driver would have control over the NIC. I think DPDK should create and DCB activated PMDport, and then somehow I need to configure the mapping. If you believe what I'm saying doesn't make any sense let me know!

I believe the source code of the `testpmd` would be a good point to start with. I track the code and found the code snippet related to the `DCB` configuration. I'm gonna port the snippet to my BESS PMDport. [Here](https://github.com/DPDK/dpdk/blob/3be76aa9294f3788b4f9c615642e6027f1b7948a/app/test-pmd/testpmd.c#L3210) is the pointer to the snippet.

### Mellanox NICs
Mellanox PMD port has some dependecies, so DPDK deactivate them by default. For resovling those dependencies. We need to install [OFED](https://www.mellanox.com/page/products_dyn?product_family=26&mtag=linux_sw_drivers&ssn=v2fobgocpc49s8m2asdsf74di0) packages which is provided by the company. However, I just realized that the current BESS only works with OFED 4.4 and it is not compatible with OFED 4.7 which is the latest version. The important thing in OFED installation, for dpdk obviously, is actually the installation parameters which are `./mlnxofedinstall --dpdk --upstream-libs`.

### Intel NICs
For Intel NICs, I have only tried `Intel 82599ES 10-Gigabit` and the only important point in the configuration would be the number of queues that should be exactly 128 when calling `rte_eth_dev_configure`. I have never had a succesfull configuration when had less than 128 queues.

There is a note at this [page](https://doc.dpdk.org/guides/prog_guide/poll_mode_drv.html) that I found it a bit late, but says: **When configuring for DCB operation, at port initialization, both the number of transmit queues and the number of receive queues must be set to 128**.
