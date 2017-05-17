```erlang
-module(e1).
-export([server/1]).
-export([client/2, test/0]).

-define(PACKET, 4).
-define(TCP_OPTIONS, [binary, {active, false}, 
                      {packet, ?PACKET},
                      {reuseaddr, true}] ).

server(Port) ->
    {ok, ServerSocket} = gen_tcp:listen(Port, ?TCP_OPTIONS),
    accept(ServerSocket).

accept(ServerSocket) ->
    {ok, ClientSocket} = gen_tcp:accept(ServerSocket),  %This is the 'controlling process', but because
    spawn(fun() -> loop(ClientSocket) end),             %messages are not landing in the mailbox,
    accept(ServerSocket).                               %{active, false}, any process can read 
                                                        %the message by calling gen_tcp:recv().
loop(ClientSocket) ->
    io:format("loop(): ~w~n", [ClientSocket]),

    case gen_tcp:recv(ClientSocket, 0) of
        {ok, Data} ->
            io:format("loop(): Data (bin): ~w~n", [Data]),
            io:format("loop(): Data (term): ~s~n", [binary_to_term(Data)]),

            gen_tcp:send(ClientSocket, Data),
            loop(ClientSocket);
        {error, closed} ->
            io:format("loop(): error, closed~n");
        Other -> 
            io:format("loop(): Other: ~w~n", [Other])
    end.


start_server(Port) ->
    spawn(?MODULE, server, [Port]).

client(Port, Str) ->
    {ok, Socket} = gen_tcp:connect("localhost", Port, ?TCP_OPTIONS), 
    ok = gen_tcp:send(Socket, term_to_binary(Str)),

    {ok, Bin} = gen_tcp:recv(Socket, 0),
    io:format("client(): Bin: ~w~n", [Bin]),
    ReturnedStr = binary_to_term(Bin),
    io:format("client(): received: ~s~n", [ReturnedStr]),
    io:format("client(): closing socket...~n"),

    gen_tcp:close(Socket).

                                   
test() ->
    Port = 25100,
    Server = start_server(Port),

    Str = "hello",
    Client = client(Port, Str),

    exit(Server, kill).
```
