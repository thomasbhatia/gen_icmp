
gen\_icmp aspires to be a simple interface for using ICMP sockets in
Erlang, just like gen\_tcp and gen\_udp do for their protocol types;
incidentally messing up Google searches for whomever someday writes a
proper gen\_icmp module.

gen\_icmp uses procket to get a raw socket and abuses gen\_udp for the
socket handling. gen\_icmp should work on Linux and BSDs.

The interfaces, return values and code may still change before the final
version. If you just need a simple example of sending a ping, also see:

<https://github.com/msantos/procket/blob/master/src/icmp.erl>


## EXPORTS

    open() -> {ok, Socket} | {error, Error}
    open(RawOptions, SocketOptions) -> {ok, Socket} | {error, Error}
    
        Types   Socket = pid()
                RawOptions = [ RawOption | {setuid,boolean()} ]
                RawOption = options()
                SocketOptions = SocketOpt
                SocketOpt = [ {active, true} | {active, once} |
                    {active, false} ]
    
        See the procket README for the raw socket options. If the
        {setuid,false} option is used, gen_icmp will not fork the setuid
        helper program to open the raw socket.  The Erlang VM will require
        the appropriate privileges to obtain the socket. For example, running
        as root or on Linux granting the CAP_NET_RAW capability to beam:
    
            setcap cap_net_raw=ep /usr/local/lib/erlang/erts-5.8.3/bin/beam.smp
    
        Only the owning process will receive ICMP packets (see
        controlling_process/2 to change the owner). The process owning the
        raw socket will receive all ICMP packets sent to the host.
    
        Messages sent to the controlling process are:
    
        {icmp, Socket, Address, Packet}
    
        Where Socket is the pid of the gen_icmp process, Address is a tuple
        representing the IPv4 source address and Packet is the complete
        ICMP packet including the ICMP headers.
    
    close(Socket) -> ok | {error, Reason}
    
        Types   Socket = pid()
                Reason = posix()
    
        Close the ICMP socket.
    
    send(Socket, Address, Packet) -> ok | {error, Reason}
    
        Types   Socket = pid()
                Address = tuple()
                Packet = binary()
                Reason = not_owner | posix()
    
        Like the gen_udp and gen_tcp modules, any process can send ICMP
        packets but only the owner will receive the responses.
    
    recv(Socket, Length) -> {ok, {Address, Packet}} | {error, Reason}
    recv(Socket, Length, Timeout) -> {ok, {Address, Packet}} | {error, Reason}
    
        Types   Socket = socket()
                Length = int()
                Address = ip_address()
                Packet = [char()] | binary()
                Timeout = int() | infinity
                Reason = not_owner | posix()
    
        This function receives a packet from a socket in passive mode.
    
        The optional Timeout parameter specifies a timeout in
        milliseconds. The default value is infinity .
    
    controlling_process(Socket, Pid) -> ok
    
        Types   Socket = pid()
                Pid = pid()
    
        Change the process owning the socket. Allows another process to
        receive the ICMP responses.
    
    setopts(Socket, Options) ->
    
        Types   Socket = pid()
                Options = list()
    
        For options, see the inet man page. Simply calls inet:setopts/2
        on the gen_udp socket.
    
    ping(Host) -> Responses
    ping(Host, Options) -> Responses
    ping(Socket, Hosts, Options) -> Responses
    
        Types   Socket = pid()
                Host = Address | Hostname | Hosts
                Address = tuple()
                Hostname = string()
                Hosts = [ tuple() | string() ]
                Options = [ Option ]
                Option = {id, Id} | {sequence, Sequence} | {timeout, Timeout} | {data, Data} |
                    {timestamp, boolean()}
                Id = uint16()
                Sequence = uint16()
                Timeout = int() 
                Data = binary()
                Responses = [ Response ]
                Response = {ok, Address, {Id, Sequence, Elapsed, Payload}} |
                    {{error, Error}, Address, {Id, Sequence, Payload}} | {{error, timeout}, Address}
                Elapsed = int()
                Payload = binary()
                Error = unreach_host | timxceed_intrans
    
        Convenience function to send a single ping packet. The argument to
        ping/1 can be either a hostname or a list of hostnames.
    
        The ping/3 function blocks until either an ICMP ECHO REPLY is received
        from all hosts or Timeout is reached.
    
        Id and sequence are used to differentiate ping responses. Usually,
        the sequence is incremented for each ping in one run.
    
        A list of responses is returned. If the ping was successful, the
        elapsed time is included (calculated by subtracting the current time
        from the time we sent in the ICMP ECHO packet and returned to us in
        the ICMP ECHOREPLY payload).
    
        The timeout is set for all ICMP packets and is set after all packets
        have been sent out.
    
        ping/1 and ping/2 open and close an ICMP socket for the user. For
        best performance, ping/3 should be used instead, with the socket
        being maintained between runs.
    
        Duplicate hosts in the address list are removed.
    
        A ping payload contains a 12 byte timestamp generated using
        erlang:now/0. When creating a custom payload, the first 12 bytes of
        the ICMP echo reply payload will be used for calculating the elapsed
        time. To disable this behaviour, use the option {timestamp,false}
        (the elapsed time in the return value will be set to 0).
    
        The timeout defaults to 5 seconds.
    
    echo(Id, Sequence) -> Packet
    
        Types   Id = uint16()
                Sequence = uint16()
                Packet = binary()
    
        Creates an ICMP echo packet with the results of erlang:now() used
        as the timestamp and a payload consisting of ASCII 32 to 75.
    
    echo(Id, Sequence, Payload) -> Packet
    
        Types   Id = uint16()
                Sequence = uint16()
                Payload = binary()
                Packet = binary()
    
        Creates an ICMP echo packet with the results of erlang:now() used
        as the timestamp and a user specified payload (which should pad the
        packet to 64 bytes).
    
    
    packet(Header, Payload) -> Packet
    
        Types   Header = [ #icmp{} | Options ]
                Options = [ Opts ]
                Opts = [{type, Type} | {code, Code} | {id, Id} | {sequence, Sequence} |
                            {gateway, Gateway} | {mtu, MTU} | {pointer, Pointer} |
                            {ts_orig, TS_orig} | {ts_recv, TS_recv} | {ts_tx, TS_tx} ]
                Type = uint8() | ICMP_type
                Code = uint8() | ICMP_code
                ICMP_type = echoreply | dest_unreach | source_quench | redirect | echo |
                    time_exceeded | parameterprob | timestamp | timestampreply | info_request |
                    info_reply | address | addressreply
                ICMP_code = unreach_net | unreach_host | unreach_protocol | unreach_port |
                    unreach_needfrag | unreach_srcfail | redirect_net | redirect_host |
                    redirect_tosnet | redirect_toshost | timxceed_intrans | timxceed_reass
                Id = uint16()
                Sequence = uint16()
                Payload = binary()
                Packet = binary()
    
        Convenience function for creating arbitrary ICMP packets. This
        function will calculate the ICMP checksum and insert it into the
        packet.


## COMPILING

    $ make

Also see the README for procket for additional setup (the procket
executable needs superuser privileges).


## EXAMPLE

### Simple ping interface

    1> gen_icmp:ping("www.yahoo.com").
    [{ok,{67,195,160,76},
         {{38223,0,47888},
          <<" !\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJK">>}}]
    
    2> gen_icmp:ping(["www.yahoo.com", {192,168,213,4}, "193.180.168.20", {192,0,32,10}]).
    
    [{{error,unreach_host},
      {192,168,213,4},
      {{2882,0},
      <<69,0,0,84,0,0,64,0,64,1,14,220,192,168,213,119,192,
        168,213,4,8,0,29,...>>}},
     {ok,{193,180,168,20},
         {{2882,0,126078},
          <<" !\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJK">>}},
     {ok,{192,0,32,10},
         {{2882,0,94281},
          <<" !\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJK">>}},
     {ok,{69,147,125,65},
         {{2882,0,29866},
          <<" !\"#$%&'()*+,-./0123456789:;<=>?@ABCDEFGHIJK">>}}]

### Re-using the ICMP ping socket

Keeping the ICMP socket around between runs is more efficient:

    {ok, Socket} = gen_icmp:open(),
    P1 = gen_icmp:ping(Socket, [{10,1,1,1}, "www.google.com"], []),
    P2 = gen_icmp:ping(Socket, [{10,2,2,2}, "www.yahoo.com"], []),
    gen_icmp:close(Socket).

To avoid forking the setuid binary (beam needs the appropriate
capabilities):

    {ok, Socket} = gen_icmp:open([{setuid,false}], []),
    P1 = gen_icmp:ping(Socket, [{10,1,1,1}, "www.google.com"], []),
    P2 = gen_icmp:ping(Socket, [{10,2,2,2}, "www.yahoo.com"], []),
    gen_icmp:close(Socket).


### Working with ICMP sockets

    {ok, Socket} = gen_icmp:open().
    
    % ICMP host unreachable, empty payload (should contain an IPv4 header
    % and the first 8 bytes of the packet data)
    Packet = gen_icmp:packet([{type, 3}, {code, 0}], <<0:160, 0:64>>).
    
    gen_icmp:send(Socket, {127,0,0,1}, Packet).


### Setting Up an ICMP Ping Tunnel

ptun is an example of using gen\_icmp to tunnel TCP over ICMP.

Host1 (1.1.1.1) listens for TCP on port 8787 and forwards the data
over ICMP:

    erl -noshell -pa ebin deps/*/ebin -eval 'ptun:server({2,2,2,2},8787)'

Host2 (2.2.2.2) receives ICMP echo requests and opens a TCP connection
to 127.0.0.1:22:

    erl -noshell -pa ebin deps/*/ebin -eval 'ptun:client({1,1,1,1},22)'

To use the proxy on host1:

    ssh -p 8787 127.0.0.1


### TODO

* support for IPv6 ICMP
