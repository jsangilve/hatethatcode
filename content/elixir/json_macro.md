Title: A macro to query over multiple fields of a JSONB column
Date: 2019-01-08
Tags: elixir, ecto, jsonb, postgresql
Category: elixir
Authors: José San Gil


When working with an `Ecto.Schema` that contains a JSONB column, you normally create a [fragment](https://hexdocs.pm/ecto/Ecto.Query.API.html#fragment/1) to query the attributes of a column.

For instance, let's assume we have the following schema:


```elixir
 defmodule Vehicle do
   @primary_key {:id, :id, autogenerate: true}
   schema "vehicle" do
     field :brand, :string
     field :model, :string
     field :specs, :map
   end
 end
```

The field `specs` is a Postgres' `jsonb` column storing vehicle's information as engine type, horsepower, dimensions, passenger capacity, etc. Let's imagine we decided to use `jsonb` because specs between vehicles can be significantly different i.e. some `jsonb` attributes of a vehicle might not exist for another model.


## Queries

If we want to get all the vehicles which brand is Mercedes-Benz and passenger capacity is 5, we would write something like this:

```elixir
 # using jsonb containment operator
 def query_by_brand_capacity(brand, capacity) do
   Vehicle
   |> where([v], v.brand == ^brand)
   |> where([v], fragment(
     "specs @> ?::jsonb", 
     ^%{"passenger_capacity" => capacity}
   ))
 end
 
 # or we could also write it using the object field operator
 def query_by_brand_capacity_v2(brand, capacity) do
   Vehicle
   |> where([v], v.brand == ^brand)
   |> where([v], fragment("(specs -> 'passenger_capacity') = ?", ^capacity))
 end
 
 # we could use this queries as follows
 iex> query_by_brand_capacity("Mercedes-Benz", 5) |> Repo.all()
 [
   %JsonbMacroExample.Schemas.Vehicle{
     __meta__: #Ecto.Schema.Metadata<:loaded, "vehicle">,
     brand: "Mercedes-Benz",
     id: 1,
     model: "E Class",
     specs: %{"passenger_capacity" => 5...}
   }
 ]
```

This is a fairly simple query, but it's restricted to the `passenger_capacity` field. If we want to query other `jsonb` fields — or vehicle's specs — we need to add them to the query function or create a new one. This doesn't scale; We don't know all the possible vehicle's specs in advance.

## A Macro

Let's say we want to have all the vehicles which `passenger_capacity` is `x`, `engine_type` is `e_type`, and `drive_type` is `d_type`.

Following the previous approach, we would define the following query:

```elixir
 def query_by_engine_capacity_drive(engine, capacity, drive) do
   Vehicle
   |> where([_v], fragment(
     "specs @> ? AND specs @> ? AND specs @> ?",
     ^%{"engine_type" => engine},
     ^%{"passenger_capacity" => capacity},
     ^%{"drive_type" => drive}
   ))
 end
```

The ideal API would provide a function where multiple `jsonb` fields can be query and combined at the same time using `AND` / `OR` operators:

```elixir
 iex> query_specs(
   query, 
   engine_type: "Electric", passenger_capacity: 5, drive_type: "Front Wheel"
 ) |> Repo.all
 [
   %JsonbMacroExample.Schemas.Vehicle{
     __meta__: #Ecto.Schema.Metadata<:loaded, "vehicle">,
     brand: "Mercedes-Benz",
     id: 1,
     model: "E Class",
     specs: %{
       "passenger_capacity" => 5, 
       "engine_type" => "Electric", 
       "drive_type" => "Front Wheel"
     }
   }
   ...
 ]
```


Let's try to define a macro that does that:

```elixir
 defmacro json_multi_expressions_v0(col, params, opts) do
   # conjuctive operator to be used between fragments
   conjuntion = Keyword.get(opts, :conjuntion, :and)
 
   quote do
     fragments =
       Enum.reduce(unquote(params), nil, fn {key, val}, acc ->
 				# create a fragment and wrap it within a Ecto's dynamic
 			  # expression that could be joined (nested).
         frag =
           dynamic(
             [q],
             fragment(
               "?::jsonb @> ?::jsonb",
               field(q, ^unquote(col)),
               ^%{to_string(key) => val}
             )
           )
 
         JsonbMacroExample.do_combine(frag, acc, unquote(conjunction))
       end)
   end
 end

 def do_combine(frag, acc, conjunction)
 def do_combine(frag, nil, _), do: frag
 def do_combine(frag, acc, :or), do: dynamic([q], ^acc or ^frag)
 def do_combine(frag, acc, _), do: dynamic([q], ^acc and ^frag)
```

The `macro` above compares each JSONB field in `params`, and combines it (via reduce) on dynamic expressions joined by `:or` or `:and` (depending on the `conjuction` option).

Using the macro above we're now able to write `query_specs/3`:

```elixir
 @spec query_specs(Ecto.Query.t(), Keyword.t()) :: [vehicle]
 def query_specs(query, params, conjunction // :and) do
 	query
 	|> where(
    [_q], 
    ^json_multi_expressions_v0(:specs, params, [conjunction: conjunction])
  )
 end
```

This works reasonably well, and allows to query as many Vehicle `specs` as necessary without having to write a new query. However, what if some `jsonb` fields require a more complex comparison?

Imagine we need to query for vehicles which `passenger_capacity` is **greater or equal than** `x`, `engine_type` is `e_type`, and `drive_type` is `d_type`. The macro above won't be enough to do this. Let's get back to the macro's implementation and introduce a new option.

First, let's make the macro more readable:

```elixir
 defmacro json_multi_expressions_v1(col, params, opts) do
   # conjuctive operator to be used between fragments
   conjunction = Keyword.get(opts, :conjunction, :and)
 
   quote do
     JsonbMacroExample.build_fragments(
       unquote(params),
       unquote(col),
       unquote(conjunction)
     )
   end
 end
 
 @doc false
 @spec build_expressions(map() | Keyword.t(), atom(), atom()) :: Macro.t()
 def build_expressions(params, col, conjunction) do
   Enum.reduce(params, nil, fn {key, val}, acc ->
     frag =
       dynamic(
         [q],
         fragment(
           "?::jsonb @> ?::jsonb",
           field(q, ^col),
           ^%{to_string(key) => val}
         )
       )
 
     JsonbMacroExample.do_combine(frag, acc, conjunction)
   end)
 end
 
 @doc false
 def do_combine(frag, acc, conjunction)
 def do_combine(frag, nil, _), do: frag
 def do_combine(frag, acc, :or), do: dynamic([q], ^acc or ^frag)
 def do_combine(frag, acc, _), do: dynamic([q], ^acc and ^frag)
```

Now each fragment is built and combined by the `build_expressions/3` function, and we just need to call this function using the fully qualified name. 

Second, let's expand `build_expressions/3` to allow a custom query expression (a custom fragment) for each `jsonb` column:


```elixir
 @doc false
 @spec build_expressions(map() | Keyword.t(), atom(), atom(), fun()) 
  :: Macro.t()
 def build_expressions(params, col, conjunction, dynamic_fun \\ nil)

 def build_expressions(params, col, conjunction, dynamic_fun) do
   Enum.reduce(params, nil, fn {key, val}, acc ->
     frag = JsonbMacroExample.build_fragment(
       col, to_string(key), val, dynamic_fun
     )

     JsonbMacroExample.do_combine(frag, acc, conjunction)
   end)
 end

 @doc false
 @spec build_fragment(binary(), atom | binary(), term(), fun()) :: Macro.t()
 def build_fragment(col, key, val, nil) do
   # build default dynamic fragment
   dynamic(
     [q],
     fragment(
       "?::jsonb @> ?::jsonb",
       field(q, ^col),
       ^%{key => val}
     )
   )
 end

 def build_fragment(col, key, val, dynamic_fun) do
   result = dynamic_fun.(col, key, val)
   case result do
     nil ->
       build_fragment(col, key, val, nil)

     _ ->
       result
   end
 end
```

We added a new parameter to `build_expressions/4` called `dynamic_fun`. This an **optional** function that should return an Ecto's `dynamic` expression. When `dynamic_fun` is nil or returns null for a given `jsonb` field, a default expression using the `jsonb` containment operator would be placed instead.

Let's get back to the query. We wanted vehicles which `passenger_capacity` is **greater or equal than** `x`, `engine_type` is `e_type`, and `drive_type` is `d_type`. We can a create custom expression to query only those vehicles which minimum capacity is greater or equal than `x`. Let's write the expression and modify our `query_specs/3` function as follows:

```elixir
 # vehicle's capacity is greater or equal than `val`
 def query_json_col(col, "passenger_capacity", val) do
   dynamic([q], fragment(
     "(?::jsonb ->> 'passenger_capacity')::int >= ?", 
     field(q, ^col), 
     ^val)
   )
 end
 
 # return nil for any other jsonb field
 def query_json_col(_, _, _), do: nil
 
 @spec query_specs(Ecto.Query.t(), Keyword.t()) :: [vehicle]
 def query_specs(query, params, conjunction \\ :and) do
 	fragments =
 		json_multi_expressions_v2(:specs, params,
 			conjunction: conjunction,
 			dynamic_fun: &query_json_col/3
 		)
 
 	query
 	|> where([_q], ^fragments)
 end
```

Using pattern matching, we created a custom expression for `passenger_capacity`. This is particularly useful for `jsonb` fields that require a bit more sophisticated expressions. For example, if we have a Vehicle's spec called `heated_seats` that usually only high-end cars include, we would check for this spec in the following say:

- A vehicle has heated seats when the value of `heated_seats` is `true`: 
```SQL
 specs @> '{"heated_seats": true}'
```
- A vehicle doesn't have heated seats when the value is false, null or the field doesn't exist:
```SQL
 (specs @> '{"heated_seats": false}') OR
 (specs @> '{"heated_seats": null}') OR 
 NOT (specs ? 'heated_seats')
```

We could write this in Elixir as follows:

```elixir
 # no heated_seats means false, nil o inexistent `heated_seats` field
 def query_json_col(:specs, "heated_seats", false) do
   dynamic(
     [q],
     fragment(
       "(specs @> ?::jsonb OR specs @> ?::jsonb OR NOT (specs \\? ?)",
       %{"heated_seats" => false},
       %{"heated_seats" => nil}
       "heated_seats",
     )
   )
 end
 
 def query_json_col(_, _, _), do: nil
```

Without changing anything at `query_specs/3`, we can now query for vehicles which `passenger_capacity` is **greater or equal than** `x`, `engine_type` is `e_type`, `drive_type` is `d_type` and `heated_seats` are `a_boolean`.

```elixir
 iex> query_specs(query, %{
   engine_type: "Electric", 
   passenger_capacity: 5, 
   drive_type: "Front Wheel",
   heated_seats: false
 }) |> Repo.all
 [
   %JsonbMacroExample.Schemas.Vehicle{
     __meta__: #Ecto.Schema.Metadata<:loaded, "vehicle">,
     brand: "Mercedes-Benz",
     id: 1,
     model: "E Class",
     specs: %{
       "passenger_capacity" => 5,
       "engine_type" => "Electric",
       "drive_type" => "Front Wheel"
     }
   }
   ...
 ]
```

Because `query_specs/3` is composable, we can do things like:

```elixir
 def family_electric_mercedes do
   Vehicle
   |> where([v], v.brand == ^brand)
   |> query_specs(%{
     passenger_capacity: 4, 
     passenger_doors: 4, 
     engine_type: "Electric"
   })
 end
```

## The final macro

This is the final `json_multi_expressions` macro that make this work:

```elixir
 @doc """
 A macro that generates multiple fragments using dynamic expressions.

 Options

   * `conjunction`: define whether to use `and` or `or` to join dynamic 
   expressions for each parameter. By default uses `and`.
   * `gen_dynamic` - An optional function to generate a dynamic fragment
   (using `Ecto.Query.dynamic`).
 """
 defmacro json_multi_expressions(col, params, opts) do
   # conjuctive operator to be used between fragments
   conjunction = Keyword.get(opts, :conjunction, :and)
   # a function that generates a dynamic expression
   dynamic_fun = Keyword.get(opts, :dynamic_fun)

   quote do
     JsonbMacroExample.build_expressions(
       unquote(params),
       unquote(col),
       unquote(conjunction),
       unquote(dynamic_fun)
     )
   end
 end

 @doc false
 @spec build_expressions(map() | Keyword.t(), atom(), atom(), fun()) 
   :: Macro.t()
 def build_expressions(params, col, conjunction, dynamic_fun \\ nil)

 def build_expressions(params, col, conjunction, dynamic_fun) do
   Enum.reduce(params, nil, fn {key, val}, acc ->
     frag = JsonbMacroExample.build_fragment(
       col, 
       to_string(key), 
       val, 
       dynamic_fun
     )

     # TODO I'd write this using a case, but it generates a compilation warning
     # https://github.com/elixir-lang/elixir/issues/6738
     JsonbMacroExample.do_combine(frag, acc, conjunction)
   end)
 end

 @doc false
 @spec build_fragment(binary(), atom | binary(), term(), fun()) :: Macro.t()
 def build_fragment(col, key, val, nil) do
   # build default dynamic fragment
   dynamic(
     [q],
     fragment(
       "?::jsonb @> ?::jsonb",
       field(q, ^col),
       ^%{key => val}
     )
   )
 end

 def build_fragment(col, key, val, dynamic_fun) do
   result = dynamic_fun.(col, key, val)

   case result do
     nil ->
       build_fragment(col, key, val, nil)

     _ ->
       result
   end
 end

 @doc false
 def do_combine(frag, acc, conjunction)
 def do_combine(frag, nil, _), do: frag
 def do_combine(frag, acc, :or), do: dynamic([q], ^acc or ^frag)
 def do_combine(frag, acc, _), do: dynamic([q], ^acc and ^frag)
```

## Notes & References

- The source code could be found in [GitHub](https://github.com/jsangilve/ecto_jsonb_macro_example)

Some good related articles and PostgreSQL docs on `jsonb`:

- [Querying an Embedded Map...](https://robots.thoughtbot.com/querying-embedded-maps-in-postgresql-with-ecto)
- [Using PostgreSQL Jsonb columns in Ecto](http://www.ubazu.com/using-postgres-jsonb-columns-in-ecto)
- [JSON Types](https://www.postgresql.org/docs/9.4/datatype-json.html)
- [JSON Functions and Operators](https://www.postgresql.org/docs/9.4/datatype-json.html)


