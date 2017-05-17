This is from a page no longer available (404 error):

    20bits.com/article/erlang-a-generalized-tcp-server

I retrieved the page using google's cache of the page.  This is not my code.

-------

## Erlang: A Generalized TCP Server

#### by Jesse Farmer on Monday, February 20, 2012

In my last few articles about Erlang we've covered the basics of network programming with gen_tcp and Erlang/OTP's gen_server, or generic server, module. Let's combine the two.

In most people's minds "server" means network server, but Erlang uses the terminology in the most abstract sense. gen_server is really a server that operates using Erlang's message passing as its base protocol. We can graft a TCP server onto that framework, but it requires some work.

#### The Structure of a Network Server
Most network servers have a similar architecture. First they create a listening socket that listens for incoming connection. They then enter an accept state in which they loop until termination, accepting each new connection as it arrives and starting the real client/server work.

To see this in action recall the simple echo server from my network programming article:
```erlang
-module(echo).
-author('Jesse E.I. Farmer <jesse@20bits.com>').
-export([listen/1]).

-define(TCP_OPTIONS, [binary, {packet, 0}, {active, false}, {reuseaddr, true}]).

% Call echo:listen(Port) to start the service.
listen(Port) ->
    {ok, LSocket} = gen_tcp:listen(Port, ?TCP_OPTIONS),
    accept(LSocket).

% Wait for incoming connections and spawn the echo loop when we get one.
accept(LSocket) ->
    {ok, Socket} = gen_tcp:accept(LSocket),
    spawn(fun() -> loop(Socket) end),
    accept(LSocket).

% Echo back whatever data we receive on Socket.
loop(Socket) ->
    case gen_tcp:recv(Socket, 0) of
        {ok, Data} ->
            gen_tcp:send(Socket, Data),
            loop(Socket);
        {error, closed} ->
            ok
    end.
```

[**Edit by 7stud**: I think the combination of `{active, false}`, i.e. messages streaming through the socket will not land in the mailbox; and `{packet, 0}`, i.e. messages will not be preceded by their length; and `gen_tcp:recv(Socket, 0)` in the example, is error prone.  I think that in order to guarantee that the server has read the whole message, the server has to loop over `gen_tcp:recv(Socket, 0)` and gather up the chunks of the message that are streaming to the socket, and the only way the server will know that it has reached the end of a message is if the client closes the socket.  To make things simpler, I think the server should specify `{packet, 4}`.  In that case, I think `gen_tcp:recv(Socket, 0)` will automatically return the entire message.  I think internally erlang will automatically read the first 4 bytes from the socket, which will tell erlang the Length of the message, then erlang will wait until it has read Length bytes from the stream, then `gen_tcp:recv(Socket, 0)` will return the entire message.  In other words, if the server specifies `{packet, 4}` then erlang will automatically do the looping necessary to receive the whole message for you, which is pretty neat.]

As you can see, listen creates a listening socket and immediately calls accept. This waits for an incoming connection, spawns a new worker (loop) that does the real work, and then waits for the next incoming connection.

In this code the parent process owns both the listen socket and the accept loop. As we'll see this doesn't work so well when we try to integrate the accept/listen loop with gen_server.

#### Abstracting The Network Server
Network servers come in two parts: connection handling and business logic. As I described above the connection handling is basically the same for every network server. Ideally we'd be able to do something like
```erlang
-module(my_server).
start(Port) ->
	connection_handler:start(my_server, Port, business_logic).

business_logic(Socket) ->
	% Read data from the network socket and do our thang.
```
Let's go ahead and do just this.

#### Implementing A Generic Network Server
The problem with implementing a network server using gen_server is that the call to gen_tcp:accept is blocking. If we were to call this in the server's initialization routine, for example, the whole gen_server mechanism would block until a client connected.

There are two ways to get around this. One involves using a lower-level connection mechanism that supports non-blocking (or asynchronous) accepting. There are then a whole family of functions, most notably gen_tcp:controlling_process, that helps you manage who receives what messages when clients connect.

A simpler and, in my opinion, more elegant solution is to have a single process that owns the listening socket. This process does two things: spawns new acceptors and listens for "connection received" messages. When it receives a message it knows to spawn a new acceptor.

An acceptor is free to call the blocking gen_tcp:accept since it's running in its own process. When it receives a connection it fires an asynchronous message back to the parent process and immediately calls the business logic function.

Here's the code. I've commented where appropriate, so hopefully it's readable.
```erlang
-module(socket_server).
-author('Jesse E.I. Farmer <jesse@20bits.com>').
-behavior(gen_server).

-export([init/1, code_change/3, handle_call/3, handle_cast/2, handle_info/2, terminate/2]).
-export([accept_loop/1]).
-export([start/3]).

-define(TCP_OPTIONS, [binary, {packet, 0}, {active, false}, {reuseaddr, true}]).

-record(server_state, {
		port,
		loop,
		ip=any,
		lsocket=null}).

start(Name, Port, Loop) ->
	State = #server_state{port = Port, loop = Loop},
	gen_server:start_link({local, Name}, ?MODULE, State, []).

init(State = #server_state{port=Port}) ->
	case gen_tcp:listen(Port, ?TCP_OPTIONS) of
   		{ok, LSocket} ->
   			NewState = State#server_state{lsocket = LSocket},
   			{ok, accept(NewState)};
   		{error, Reason} ->
   			{stop, Reason}
	end.

handle_cast({accepted, _Pid}, State=#server_state{}) ->
	{noreply, accept(State)}.

accept_loop({Server, LSocket, {M, F}}) ->
	{ok, Socket} = gen_tcp:accept(LSocket),
	% Let the server spawn a new process and replace this loop
	% with the echo loop, to avoid blocking 
	gen_server:cast(Server, {accepted, self()}),
	M:F(Socket).
	
% To be more robust we should be using spawn_link and trapping exits
accept(State = #server_state{lsocket=LSocket, loop = Loop}) ->
	proc_lib:spawn(?MODULE, accept_loop, [{self(), LSocket, Loop}]),
	State.

% These are just here to suppress warnings.
handle_call(_Msg, _Caller, State) -> {noreply, State}.
handle_info(_Msg, Library) -> {noreply, Library}.
terminate(_Reason, _Library) -> ok.
code_change(_OldVersion, Library, _Extra) -> {ok, Library}.
```

We use gen_server:cast to pass asynchronous messages back to the listening process. When the listening process receives the message accepted it spawns a new acceptor.

Right now this server is not very robust because if the active acceptor fails, for whatever reason, the server will stop accepting connections. To make it more OTP-like we should be trapping exits and firing off a new acceptor in the event that a connection fails.

#### A "Generic" Echo Server
The echo server is the easiest server to write, so let's do it using our new abstract socket server.
```erlang
-module(echo_server).
-author('Jesse E.I. Farmer <jesse@20bits.com>').

-export([start/0, loop/1]).

% echo_server specific code
start() ->
	socket_server:start(?MODULE, 7000, {?MODULE, loop}).
loop(Socket) ->
    case gen_tcp:recv(Socket, 0) of
        {ok, Data} ->
            gen_tcp:send(Socket, Data),
            loop(Socket);
        {error, closed} ->
            ok
    end.
```

As you can see the "server" becomes nothing more than its business logic. The connection handling has been generalized and pushed off into its own socket_server. The loop in our generic server is actually identical to the loop in our original echo server, too.

Hopefully you all can learn from this as much as I did. I finally feel like I'm starting to understand Erlang.

Also, feel free to leave a comment, especially if you have any thoughts on how I can improve my code. Cheers!

