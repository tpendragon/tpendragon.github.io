---
layout: post
title: Profiling Elixir
tags: elixir profiling benchmarking n-triples
---

I just started playing with [Elixir](http://elixir-lang.org/) and I've been very
impressed with the tooling that exists for it, already. One of my first projects
was getting a very basic N-Triples Parser working, and I wanted to document how
I went about improving its performance.

## Initial Implementation

I'm going to skip through to the first implementation, and not talk much about
how the code works. It basically just splits n-triples by newlines, runs regex
on each line, and then organizes all the statements into a big Map.

```elixir
defmodule NTriples.Parser do
  alias RDF.Literal
  @ntriples_regex ~r/(?<subject><[^\s]+>|_:([A-Za-z][A-Za-z0-9\-_]*))[
  ]*(?<predicate><[^\s]+>)[
  ]*(?<object><[^\s]+>|_:([A-Za-z][A-Za-z0-9\-_]*)|"(?<literal_string>(?:\\"|[^"])*)"(@(?<literal_language>[a-z]+[\-A-Za-z0-9]*)|\^\^<(?<literal_type>[^>]+)>)?)[ ]*./i
  def parse(content) do
    content
    |> String.split(".\n")
    |> Enum.filter(fn(str) -> String.length(str) > 0 end)
    |> Enum.map(&capture_triple_map/1)
    |> Enum.reduce(%{}, &process_capture/2)
  end

  defp process_capture(nil, accumulator) do
    accumulator
  end

  defp capture_triple_map("") do
    %{}
  end

  defp capture_triple_map(string) do
    Regex.named_captures(@ntriples_regex, string)
  end

  defp process_capture(map, accumulator) when map == %{} do
    accumulator
  end

  defp process_capture(capture_map, accumulator) do
    subject = process_subject(capture_map["subject"])
    predicate = process_subject(capture_map["predicate"])
    object = process_object(capture_map)
    append_triple(accumulator, {subject, predicate, object})
  end

  defp append_triple(map, {subject, predicate, object}) do
    case map do
      %{^subject => %{^predicate => existing_value}} when is_list(existing_value) ->
        new_value = existing_value ++ [object]
        Map.put(map, subject, Map.merge(map[subject], %{predicate => new_value}))
      %{^subject => %{^predicate => existing_value}} ->
        new_value = [existing_value, object]
        Map.put(map, subject, Map.merge(map[subject], %{predicate => new_value}))
      %{^subject => existing_subject_graph} ->
        Map.put(map, subject, Map.merge(map[subject], %{predicate => object}))
      _ ->
        Map.put(map, subject, %{predicate => object})
    end
  end

  defp process_subject("<" <> subject) do
    subject
    |> String.rstrip(?>)
  end

  defp process_object(%{"object" => "<" <> object_uri}) do
    uri = process_subject("<" <> object_uri)
    %{"@id" => uri}
  end

  defp process_object(%{"literal_string" => value, "literal_language" => language}) do
      %Literal{value: value, language: language}
  end
end
```

## Initial Benchmark

Let's get an initial benchmark, using the
[Benchfella](https://github.com/alco/benchfella) tool.

1. Add the dependency to your `mix.exs` file

     ```elixir
     defp deps do
       [
         {:benchfella, "~> 0.3.0", only: [:dev, :test]}
       ]
     end
     ```
2. `mix deps.get`
3. Create a benchmark file at `bench/ntriples_bench.exs`
     ```elixir
     defmodule NTriplesBench do
       use Benchfella
       @content elem(File.read("test/fixtures/content.nt"),1)
     
       bench "parse large file" do
         NTriples.parse(@content)
       end
     end
     ```
4. Run first benchmark via `mix bench`
     ```elixir
     Settings:
       duration:      1.0 s
     
     ## NTriplesBench
     [12:28:22] 1/1: parse large file
     
     Finished in 1.47 seconds
     
     ## NTriplesBench
     parse large file           5   223998.60 µs/op
     ```

So there's our first data point: 224 ms. That's not awful, but in the context of
a web application it's not great.


## Profiling Slowness

Let's use [ExProf](https://github.com/parroty/exprof) to profile where the slow
bits are.

1. Add the dependency to mix.exs

     ```elixir
     defp deps do
       [
         {:benchfella, "~> 0.3.0", only: [:dev, :test]},
         {:exprof, "~> 0.2.0", only: [:dev, :test]}
       ]
     end
     ```
2. `mix deps.get`
3. Pop open an IEX terminal via `iex -s mix`
4. Make a little profiler at `lib/profilers/ntriples.ex`

     ```elixir
     defmodule Profilers.NTriples do
       import ExProf.Macro
       @content elem(File.read("test/fixtures/content.nt"),1)
     
       def profile do
         profile do
           run
         end
       end
     
       def run do
         NTriples.parse(@content)
       end
     end
     ```
5. Launch a shell with `iex -S mix`
6. Run it

   ```elixir
   iex(2)> Profilers.NTriples.profile
   FUNCTION                                             CALLS        %    TIME  [uS / CALLS]
   --------                                             -----  -------    ----  [----------]
   'Elixir.NTriples.Parser':parse/1                         1     0.00       0  [      0.00]
   'Elixir.Profilers.NTriples':run/0                        1     0.00       0  [      0.00]
   erlang:send/2                                            1     0.00       0  [      0.00]
   'Elixir.NTriples':parse/1                                1     0.00       1  [      1.00]
   binary:split/3                                           1     0.00       1  [      1.00]
   'Elixir.String':split/3                                  1     0.00       1  [      1.00]
   'Elixir.Enum':map/2                                      1     0.00       1  [      1.00]
   'Elixir.Enum':reduce/3                                   2     0.00       1  [      0.50]
   lists:reverse/1                                          1     0.00       2  [      2.00]
   'Elixir.String':split/2                                  1     0.00       2  [      2.00]
   'Elixir.Enum':filter/2                                   1     0.00       3  [      3.00]
   'Elixir.Profilers.NTriples':'-profile/0-fun-0-'/0        1     0.00       4  [      4.00]
   binary:get_opts_split/2                                  2     0.00       7  [      3.50]
   lists:reverse/2                                          1     0.01      60  [     60.00]
   'Elixir.Regex':named_captures/2                       6021     0.06     403  [      0.07]
   'Elixir.Keyword':'-delete/2-lists^filter/1-0-'/2      6021     0.07     449  [      0.07]
   'Elixir.Regex':named_captures/3                       6021     0.07     461  [      0.08]
   'Elixir.String.Graphemes':length/1                    6022     0.07     464  [      0.08]
   'Elixir.NTriples.Parser':capture_triple_map/1         6021     0.09     573  [      0.10]
   'Elixir.Enum':into/2                                  6021     0.10     617  [      0.10]
   'Elixir.NTriples.Parser':process_capture/2            6021     0.10     654  [      0.11]
   binary:part/2                                         6022     0.11     684  [      0.11]
   maps:put/3                                            6021     0.12     763  [      0.13]
   'Elixir.Enum':zip/2                                   6021     0.14     872  [      0.14]
   'Elixir.Enum':'-map/2-lists^map/1-0-'/2               6022     0.14     896  [      0.15]
   'Elixir.NTriples.Parser':'-parse/1-fun-1-'/1          6021     0.14     902  [      0.15]
   'Elixir.String':length/1                              6022     0.15     941  [      0.16]
   'Elixir.NTriples.Parser':'-parse/1-fun-2-'/2          6021     0.15     960  [      0.16]
   'Elixir.Enum':'-filter/2-fun-0-'/3                    6022     0.15     969  [      0.16]
   binary:do_split/5                                     6022     0.15     970  [      0.16]
   lists:keyfind/3                                      12042     0.15     988  [      0.08]
   'Elixir.NTriples.Parser':'-parse/1-fun-0-'/1          6022     0.15     993  [      0.16]
   'Elixir.Regex':names/1                                6021     0.16    1015  [      0.17]
   'Elixir.Keyword':put/3                                6021     0.16    1022  [      0.17]
   'Elixir.Regex':run/3                                  6021     0.16    1042  [      0.17]
   'Elixir.Keyword':delete/2                             6021     0.20    1286  [      0.21]
   'Elixir.Enum':'-reduce/3-lists^foldl/2-0-'/3         12045     0.21    1356  [      0.11]
   'Elixir.Access':get/3                                18061     0.22    1389  [      0.08]
   maps:from_list/1                                      6021     0.22    1432  [      0.24]
   'Elixir.NTriples.Parser':append_triple/2              6021     0.26    1668  [      0.28]
   'Elixir.String':replace_trailing/3                   18057     0.30    1910  [      0.11]
   'Elixir.Keyword':get/3                               12042     0.30    1921  [      0.16]
   'Elixir.String':rstrip/2                             18057     0.32    2029  [      0.11]
   re:inspect/2                                          6021     0.37    2367  [      0.39]
   'Elixir.NTriples.Parser':process_object/1             6021     0.39    2503  [      0.42]
   maps:find/2                                          18061     0.40    2561  [      0.14]
   'Elixir.Access':get/2                                18061     0.43    2771  [      0.15]
   'Elixir.Access':fetch/2                              18061     0.49    3170  [      0.18]
   'Elixir.NTriples.Parser':process_subject/1           18057     0.51    3313  [      0.18]
   maps:merge/2                                         12040     0.66    4259  [      0.35]
   binary:matches/3                                         1     0.67    4305  [   4305.00]
   'Elixir.Enum':do_zip/2                               42147     0.87    5593  [      0.13]
   'Elixir.String':replace_trailing/6                   36114     1.35    8676  [      0.24]
   re:run/3                                              6021     2.78   17915  [      2.98]
   erlang:'++'/2                                         6008    12.55   80808  [     13.45]
   'Elixir.String.Graphemes':do_length/2              1456345    17.98  115788  [      0.08]
   'Elixir.String.Graphemes':next_extend_size/2       1450323    18.96  122101  [      0.08]
   'Elixir.String.Graphemes':next_grapheme_size/1     1456345    36.99  238230  [      0.16]
   -------------------------------------------------  -------  -------  ------  [----------]
   Total:                                             4778436  100.00%  644072  [      0.13]
   ```

## Evaluating Results

The functions which took the largest total amount of time show up at the bottom.
In our case the easy win is here:

```elixir
   'Elixir.String.Graphemes':do_length/2              1456345    17.98  115788  [      0.08]
   'Elixir.String.Graphemes':next_extend_size/2       1450323    18.96  122101  [      0.08]
   'Elixir.String.Graphemes':next_grapheme_size/1     1456345    36.99  238230  [      0.16]
```

Each "character" in a String is a grapheme, and it's spending a LONG time
finding the length of strings. The only time we do this in the parser is here:

```elixir
    |> Enum.filter(fn(str) -> String.length(str) > 0 end)
```

so that we don't run Regex over empty strings. Let's try just removing it and
letting the regex run.

New Results:

```elixir
ex_fedora master % mix bench
Compiled lib/ntriples/parser.ex
Settings:
  duration:      1.0 s

## NTriplesBench
[12:41:05] 1/1: parse large file

Finished in 1.57 seconds

## NTriplesBench
parse large file          10   143116.70 µs/op
```

Down to 143 ms! (and the tests pass)

## Repeat

Profiling again:

```elixir
ex_fedora master % iex -S mix
Erlang/OTP 18 [erts-7.2.1] [source] [64-bit] [smp:8:8] [async-threads:10] [hipe] [kernel-poll:false] [dtrace]

Interactive Elixir (1.2.3) - press Ctrl+C to exit (type h() ENTER for help)
iex(1)> Profilers.NTriples.profile
FUNCTION                                            CALLS        %    TIME  [uS / CALLS]
--------                                            -----  -------    ----  [----------]
'Elixir.NTriples':parse/1                               1     0.00       0  [      0.00]
code:ensure_loaded/1                                    2     0.00       0  [      0.00]
binary:split/3                                          1     0.00       0  [      0.00]
erlang:send/2                                           1     0.00       0  [      0.00]
error_handler:undefined_function/3                      2     0.00       1  [      0.50]
code:call/1                                             2     0.00       1  [      0.50]
'Elixir.String':split/3                                 1     0.00       1  [      1.00]
'Elixir.String':split/2                                 1     0.00       1  [      1.00]
'Elixir.Enum':reduce/3                                  1     0.00       1  [      1.00]
'Elixir.NTriples.Parser':parse/1                        1     0.00       1  [      1.00]
'Elixir.Profilers.NTriples':run/0                       1     0.00       1  [      1.00]
erlang:function_exported/3                              2     0.00       1  [      0.50]
error_handler:ensure_loaded/1                           2     0.00       2  [      1.00]
binary:get_opts_split/2                                 2     0.00       2  [      1.00]
'Elixir.Enum':map/2                                     1     0.00       2  [      2.00]
code_server:call/2                                      2     0.00       3  [      1.50]
erlang:whereis/1                                        2     0.00       3  [      1.50]
'Elixir.Profilers.NTriples':'-profile/0-fun-0-'/0       1     0.00       4  [      4.00]
'Elixir.Regex':named_captures/2                      6021     0.22     443  [      0.07]
'Elixir.Keyword':'-delete/2-lists^filter/1-0-'/2     6021     0.22     453  [      0.08]
'Elixir.Regex':named_captures/3                      6021     0.27     555  [      0.09]
'Elixir.NTriples.Parser':capture_triple_map/1        6022     0.29     585  [      0.10]
'Elixir.Enum':into/2                                 6021     0.30     607  [      0.10]
'Elixir.NTriples.Parser':process_capture/2           6022     0.34     681  [      0.11]
'Elixir.Enum':'-reduce/3-lists^foldl/2-0-'/3         6023     0.35     704  [      0.12]
binary:part/2                                        6022     0.37     754  [      0.13]
maps:put/3                                           6021     0.41     824  [      0.14]
'Elixir.Enum':'-map/2-lists^map/1-0-'/2              6023     0.45     903  [      0.15]
'Elixir.NTriples.Parser':'-parse/1-fun-0-'/1         6022     0.45     913  [      0.15]
'Elixir.Keyword':put/3                               6021     0.46     931  [      0.15]
'Elixir.Enum':zip/2                                  6021     0.47     953  [      0.16]
lists:keyfind/3                                     12042     0.49     986  [      0.08]
'Elixir.Regex':names/1                               6021     0.50    1006  [      0.17]
binary:do_split/5                                    6022     0.50    1021  [      0.17]
'Elixir.NTriples.Parser':'-parse/1-fun-1-'/2         6022     0.56    1131  [      0.19]
'Elixir.Regex':run/3                                 6021     0.56    1141  [      0.19]
'Elixir.Keyword':delete/2                            6021     0.58    1181  [      0.20]
'Elixir.Access':get/3                               18061     0.79    1609  [      0.09]
'Elixir.NTriples.Parser':append_triple/2             6021     0.87    1769  [      0.29]
'Elixir.Keyword':get/3                              12042     0.94    1900  [      0.16]
re:inspect/2                                         6021     0.96    1949  [      0.32]
'Elixir.String':replace_trailing/3                  18057     1.05    2119  [      0.12]
'Elixir.String':rstrip/2                            18057     1.14    2315  [      0.13]
'Elixir.NTriples.Parser':process_object/1            6021     1.26    2559  [      0.43]
maps:from_list/1                                     6021     1.29    2611  [      0.43]
maps:find/2                                         18061     1.37    2780  [      0.15]
'Elixir.Access':get/2                               18061     1.58    3204  [      0.18]
'Elixir.Access':fetch/2                             18061     1.72    3485  [      0.19]
maps:merge/2                                        12040     1.84    3731  [      0.31]
'Elixir.NTriples.Parser':process_subject/1          18057     2.01    4066  [      0.23]
binary:matches/3                                        1     2.52    5102  [   5102.00]
'Elixir.Enum':do_zip/2                              42147     3.35    6781  [      0.16]
'Elixir.String':replace_trailing/6                  36114     4.62    9355  [      0.26]
re:run/3                                             6021     9.00   18222  [      3.03]
erlang:'++'/2                                        6008    55.89  113202  [     18.84]
-------------------------------------------------  ------  -------  ------  [----------]
Total:                                             385328  100.00%  202555  [      0.53]
```

Now the slowest piece is the ++ operator for combining two lists, as called
here:

```elixir
        new_value = existing_value ++ [object]
```

Lists are linked list - PREPENDING a node should be VERY fast, much quicker than
adding two lists together. Let's change that to

```elixir
        new_value = [object | existing_value]
```

(NOTE: This switches the order of the objects in the parsing from the order
they're in the file, line by line. However, those terms are explicitly
unordered, so it's technically okay for them to be reversed. If we want to, we
can reverse them later.)

Results:

```elixir
ex_fedora master % mix bench
Compiled lib/ntriples/parser.ex
Settings:
  duration:      1.0 s

## NTriplesBench
[12:44:49] 1/1: parse large file

Finished in 3.31 seconds

## NTriplesBench
parse large file          50   52658.56 µs/op
```

52 ms!

## The End

That's all there is to it. At this point if you run the profiler again, the
biggest thing taking up time is running the regular expressions. Speeding it up
at that point probably means building a real syntax parser - but for now, this
solution works.

Hope this helps!
