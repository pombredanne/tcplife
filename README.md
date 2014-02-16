tcplife
=======

## About

tcplife is a systemtap script that will help you understand 
the life of tcp connection as seen from the linux kernel.

tcplife uses sytemtap probes placed in important sections 
of the tcp be able to catch and display events occuring
during the connection.

You can then use this information to precisely understand
why a connection became slow, what is the speed limit factor, 
etc.

## Compatibility 

tcplife has currently only be tested on Red Hat Entreprise Linux 6
with kernel 2.6.32.

## Usage

No option yet, to start to monitor all tcp connection, simply launch
the script with either:
    tcplife.stp
or 
    stap tcplife.stp

## Output 

For each event, tcplife will display two lines like this:

    Source_IP:Source_Port <-> Destination_IP:Destination_Port Event: Description of the event
    Source_IP:Source_Port <-> Destination_IP:Destination_Port Status: Various information about the current state of the connection

The event part is just a descriptive line, whereas the status part contains
precise numerical information:

* `sndbuf`: information about the TCP socket buffer size in the form of the
            triplet buffer usage / buffer size / maximum buffer size, where:

  * _buffer usage_ is the number of bytes currently used by packets waiting
                   to be sent.
  * _buffer size_ is the current buffer size, this value is dynamically
                  increased by the kernel when needed during the life of the
                  tcp connection.
  * _maximum buffer size_: the maximum buffer size allowed by the kernel. 

  All values are in bytes and minmum, default and maximum values can be tuned
  using the `/proc/sys/net/ipv4/tcp_wmem` kernel parameter.
  More information can be found in the [ip-sysctl.txt][1] in the kernel
  documentation.

[1] https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/tree/Documentation/networking/ip-sysctl.txt?id=refs/tags/v2.6.32.61#n452

* `snd_wnd`: the sender window in the following form
             number of bytes sent and not acknowledged / number of bytes allowed

   This give information about the number of data that the receiver allowed us to send
   in order to let him the time to process the data.

   The number of bytes allowed and acknowledged are derived from the ack and
   window information sent by the receiver in each acknowledgement TCP packet.

* `cwnd`: information about the congestion window in the following form
          number of packets in flight / maximum packets allowed

   This give information about how much we can send through the network without 
   overflowing it, according to the TCP congestion algorithm used.
   
* `ssthresh`: slow-start threshold.

   Usually TCP congestion algorithms try to quickly increase the throughput of a
   tcp connection until that threshold is reached, after a softer and different
   increase algorithm is used.
   This is however dependent on the TCP congestion algorithm used.

* `congestion_state`: internal congestion state maintained by the Linux TCP stack

   It can take the following values:

   * _Open_    : normal mode
   * _Disorder_: some packets were received out of order by the receiver.
   * _CWR_     : congestion windows reduction mode, mode reached when a
                 congestion notification event is received.
   * _Recovery_: congestion window was reduced. 
   * _Loss_    : some packets were probably lost. 

   More information can be found in the linux [tcp_input.c source file][2].

[2] https://git.kernel.org/cgit/linux/kernel/git/stable/linux-stable.git/tree/net/ipv4/tcp_input.c?id=refs/tags/v2.6.32.61#n2321


Example of output:

    182.168.0.24:22 <-> 192.168.0.37:36433 Event: Enabling Forward RTO Recovery algorithm, changing state Open -> Disorder
    182.168.0.24:22 <-> 192.168.0.37:36433 Status: sndbuf:1776/23720/4194304, snd_wnd:496/45184 cwnd:1/2 ssthresh:7 congestion_state:Disorder



## List of events traced

### On the sending path

Here is what roughly happens when a packet is sent and what event tcplife will
show.
 
1. The data are sent via the `tcp_sendpages` or `tcp_sendmsg` functions.

   These functions will first try to allocate the memory required to store
   the data and will block if that's not possible, which happens in two cases:

  * the maximum memory allowed for tcp has been reached (*Event "TCP Memory full"*) 
  * the TCP socket send buffer is full (*Event "Send buffer full"*)
   
2. If that's ok, the data are then passed to the  tcp_write_xmit`function.

   This one is supposed to sent the packet to the network, but it will not do
   it immediately if:
  
    * the congestion window is full (*Event: "Congestion window full"*)
    * the sending window is full (*Event "Send window full"*),
    * other reasons not traced by this script because they are part of 
      the normal flow (Nagle, TSO).

   This function will not block in those cases and will simply return. 
   The packet not sent will continue to take space in the sender buffer
   while waiting to be sent.

3. If the packet can be sent, it will passed to the `tcp_transmist_skb` function;

   This function will create the TCP header and pass the packet to the lower
   layer which will be the the traffic control layer and/or the network card
   layer.

   At this point, we only detect congestion event raised by the lower layer
   without knowing if it's the netword card that is overloaded or if is the
   traffic layer which drops or slows packets because of policy (*Event
   "Local device or traffic congestion"*).
   
4. Later, when the sender receives an acknowlegdement from the receiver, it can do
   several things related to the packet sending:

   * if the acknowledgement allowed to free some space in the send buffer, the
     kernel may tries to check if send buffer can be increased through the function
     `tcp_check_space` (*Event "Send buffer increase"*).

   * the information received in the acknowledgement packet are used to trigger 
     various congestion state change through the `tcp_fastretrans_alert` function:

      - a packet with the ECE flag will move the TCP connection into CWR mode
        (*Event "Explicit Congestion Notification received"*) and
        (*Event "Entering congestion window reduction mode"*).

      - a packet containing SACK or a a duplicate ack will trigger the Recovery
        mode (*Event "Selective ack received" or "Duplicate ack received"*).

      - when the receiver ackwnlowedged the full window of missing packets during
        a congestion event, the TCP stack will go back to normal mode
        (*Event "Full window acknowledged"*).

5. If the sender never acknowledges our packet, a timer will trigger in order to
   retransmit the packet. This can lead to two importante events:

   * either the TCP stack directly enter into Loss mode
     (*Event "Retransmission timeout"*)
  
   * or it will try first to enable the F-RTO algorithm
     (*Event "Retransmission timeout, enabling Forward RTO Recovery algorithm*").
    
    
### Receive path

To be done
 


