---
tags: erlang json
---
I've been digging deeper into my experiments with Erlang and have
improved the JSON parser I discussed in a
[previous blog](https://ccpsnorlax.blogspot.is/2017/09/simple-json-parsing-in-erlang.html).
The code lives at [https://github.com/snorristurluson/erl-simple-json]()
in case you want to take a look.

The project is now set up to take advantage of [rebar3](http://www.rebar3.org/), the Erlang
build system, and tests are run automatically on
[Travis CI](https://travis-ci.org/snorristurluson/erl-simple-json).
Thanks, [Lou Xun](https://github.com/aquarhead)!

The parser now does proper tokenizing, rather than the simple
string split operations I did in my first pass. The parser itself is
surprisingly simple when it's working with tokens - here it is in
its entirety:

```erlang
parse_json(Input) ->
    {Value, <<>>} = parse_json_value(Input),
    Value.

parse_json_value(Input) ->
    {Token, Rest} = tokens:tokenize(Input),
    case Token of
        open_brace ->
            populate_object(dict:new(), Rest);
        open_square_bracket ->
            populate_list([], Rest);
        {_, Value} ->
            {Value, Rest};
        Other ->
            {Other, Rest}
    end.

populate_object(Obj, Input) ->
    {Token, Rest} = tokens:tokenize(Input),
    case Token of
        close_brace ->
            {Obj, Rest};
        {qouted_string, Field} ->
            {colon, Rest2} = tokens:tokenize(Rest),
            {Value, Rest3} = parse_json_value(Rest2),
            Obj2 = dict:store(Field, Value, Obj),
            populate_object(Obj2, Rest3);
        comma ->
            populate_object(Obj, Rest)
    end.

populate_list(List, Input) ->
    {Token, Rest} = tokens:tokenize(Input),
    case Token of
        close_square_bracket ->
            {List, Rest};
        comma ->
            populate_list(List, Rest);
        open_brace ->
            {Obj, Rest2} = populate_object(dict:new(), Rest),
            populate_list(lists:append(List, [Obj]), Rest2);
        {_, Value} ->
            populate_list(lists:append(List, [Value]), Rest);
        Other ->
            populate_list(lists:append(List, [Other]), Rest)

    end.
```

This code handles the JSON snippets I've thrown at it so far,
but I need to find a good source of JSON files to ensure it
works on any valid JSON.

This project has proven to be a good way to gain some Erlang
experience. It also fits really well for a
[TDD approach](https://en.wikipedia.org/wiki/Test-driven_development),
which is always fun. I'm still struggling with how to apply TDD
on a daily basis, working in a large (and old) codebase. Starting
with a clean slate, on a small project, TDD is a no-brainer. 