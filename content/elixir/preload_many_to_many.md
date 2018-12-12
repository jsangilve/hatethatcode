Title: Using Ecto's preload functions for Many to Many relationships
Date: 2018-12-10
Tags: elixir, ecto, postgresql
Category: elixir
Authors: José San Gil


While working on 1Bitcloud's infrastructure (an application to manage crypto mining farms), I had to solve the following problem:

A `wallet` (table 1) can be mined on different `pools`, whereas a `pool` (table 2) can be assigned to many `wallets`. Then we have a table called `wallet_pool` representing a **many** to **many** relationship between both entities. This relatioship also includes a column named `priority`, which defines the order to select the pools while _mining_ a given `wallet`. **Given one or more wallets, I needed to `preload` the related pools ordered by `priority`.**
  
The Ecto's schemas look like this:

```elixir
# Schema mapping `wallet` table
defmodule Wallet do
  @primary_key {:id, :id, autogenerate: true}
  @timestamps_opts [type: :utc_datetime]
  schema "wallet" do
    field(:address, :string)
    field(:currency, :string)
    ...  

    # many to many relationshio with schema
    many_to_many(
      :pools,
      Pool,
      join_through: "wallet_pool",
      join_keys: [wallet_id: :id, pool_id: :id]
    )
     
  end
end

# Schema mapping `pool` table
defmodule Pool do

  @timestamps_opts [type: :utc_datetime_usec]
  schema "pool" do
    field(:name, :string)
    field(:url, :string)
    field(:currency, :string)
    ...
  end
```

N.B: I only defined the `ManyToMany` relationship on `Wallet`, because that's the side of the relationship we usually query.

## Preloading the Pools

By default is very easy to preload a relationship with Ecto:

```elixir

Wallet
|> Repo.get(wallet_id)
|> Repo.preload(:pools)

# or using Ecto.Query API
from(w in Wallet, where: w.id == ^wallet_id, preload: [:pools])
|> Repo.all()
``` 

The problem with this approach was that fetched pools were not ordered by the `priority` column from `wallet_pools` table. Considering this, I gave it a try to a [preload query](https://hexdocs.pm/ecto/Ecto.Query.html#preload/3-preload-queries):

```elixir
# Preload query to fetch pools
q = from(
  p in Pool,
  join: r in "wallet_pool",
  on: p.id == r.pool_id,
  order_by: r.priority
)

# wallets_or_wallets could be a %Wallet{} struct or a list of them
wallet_or_wallets = from(w in Wallet, where: w.id in [99, 100])

wallet_or_wallets
|> Repo.preload(pools: q)
```

This resulted in the following SQL query:

```sql
SELECT p0."id", p0."name", p0."url", p0."currency", w2."id" FROM "pool" AS p0 
INNER JOIN "wallet_pool" AS w1 ON p0."id" = w1."pool_id" 
INNER JOIN "wallet" AS w2 ON w2."id" = ANY($1) 
INNER JOIN "wallet_pool" AS w3 ON w3."wallet_id" = w2."id" 
WHERE (w3."pool_id" = p0."id") ORDER BY w2."id", w1."priority" [[99, 100]]
```

**...and it doesn't work**, because ECTO seems to INNER JOIN the many to many (`wallet_pool`) relationship twice, fetching the same pools many times. I could use `distinct` to avoid filter the repeated result, but there is still the unnecessary extra `JOIN` with `wallet_pool`.

I was able to write one query that did the whole thing as follows:

```elixir
wallet_ids = [99, 100]
from(
  w in Wallet,
  join: r in "wallet_pool", on: w.id == r.wallet_id,
  join: p in Pool, on: p.id == r.pool_id,
  where: w.id in ^wallet_ids,
  preload: [pools: p],
  order_by: [w.id, r.priority]
)
|> Repo.all()
```

However, this doesn't scale easily. I wanted to wrap the `preload` query within a function to make it composable i.e., it doesn't matter how do you query the `wallet` table, as long as you have a `%Wallet{}` or a list of `%Wallet{}`, it should work.

**INTRODUCE EXAMPLE** of kinda composable function.. I was able to do something like this

That's when I checked [preload functions](https://hexdocs.pm/ecto/Ecto.Query.html#preload/3-preload-functions). When I started writing these queries, the project was using Ecto `2.2.11`. While reading about preload functions, I found this [bug](https://github.com/elixir-ecto/ecto/issues/2534) that was solved by _Valim_ for Ecto 3.0. According to the [CHANGELOG](https://github.com/elixir-ecto/ecto/blob/master/CHANGELOG.md#bug-fixes-4), the behaviour for preload functions on `many_to_many` associations was erratic in Ecto 2.x.x. I migrated to Ecto 3.0.X (which was already in our TODO list) and gave it a try.

```elixir
defmodule WalletHelper do
  @moduledoc """
  Helpers functions to query Wallet schema.
  """
  import Ecto.Query

  @doc """
  Ecto preload function: lists pools related to the given `wallet_ids`,
  ordered by the `priority` defined at `wallet_pool` association.
  """
  def preload_pools(wallet_ids) do
    Pool
    |> join(:inner, [p], r in "wallet_pool", on: r.wallet_id in ^wallet_ids and p.id == r.pool_id)
    |> order_by([_, r], r.priority)
    |> select([p, r], {r.wallet_id, p})
    |> Repo.all()
  end
end

# Calling the preload function
iex> WalletHelper.preload_pools([99, 100])
[
  {99,
     %Pool{
       id: 32,
       currency: "BTC",
       port: 25,
       url: "stratum+tcp://stratum.example.com"
    }
  },
  {99,
     %Pool{
       id: 33,
       currency: "BTC",
       port: 25,
       url: "stratum+tcp://stratum.example2.com"
    }
  },
  {100,
     %Pool{
       id: 50,
       currency: "BTC",
       port: 25,
       url: "stratum+tcp://stratum.example3.com"
    }
  }
]
```
N.B: this only works with Ecto 3.0.x.

As you can see, the preload functions returns a `tuple` which first element is the id of the related wallet, while the second one is the `pool` struct. This what the prelaod function must return according to the [docs](https://hexdocs.pm/ecto/Ecto.Query.html#preload/3-preload-functions):

> For many_to_many - the function receives the IDs of the parent association and it must return a tuple with the parent id as first element and the association map or struct as second.

Finally, using the preload function is quite simple:

```elixir
# How do you query the wallets is totally indepent of the preload function
iex> Repo.get(Wallet, 99) |> Repo.preload(pools: &WalletHelper.preload_pools/1)
 %Wallet{
    __meta__: #Ecto.Schema.Metadata<:loaded, "wallet">,
    id: 99,
    currency: "BTC",
    address: "some_bitcoin_address",
    pools: [
      %Pool{
        id: 32,
        currency: "BTC",
        port: 25,
        url: "stratum+tcp://stratum.example.com"
      },
      %Pool{
       id: 33,
       currency: "BTC",
       port: 25,
       url: "stratum+tcp://stratum.example2.com"
      }
    ]
  }
```

The query above can also be written using `Ecto.Query`. I prefer the syntax above, though.

```elixir
iex> import Ecto.Query
iex> Wallet |> where([w], w.id == 99) |> preload([pools: ^&WalletHelper.preload_pools/1]) |> Repo.all()
... same result
```

## Bonus

I wanted to include the pool's priority on the results, so I did a slight modification on Pool's schema adding a virtual field called `priority`:

```elixir
# Schema mapping `pool` table
defmodule Pool do
  @timestamps_opts [type: :utc_datetime_usec]
  schema "pool" do
    field(:name, :string)
    field(:url, :string)
    field(:currency, :string)
    ...
    # new virtual field
    field(:priority, :string, virtual: true)
  end
```

...then added the priority value within `preload_pools` function using [`Ecto.Query.merge/2`](https://hexdocs.pm/ecto/Ecto.Query.API.html#merge/2):

```elixir
defmodule WalletHelper do
  ...
  
  def preload_pools(wallet_ids) do
    Pool
    |> join(:inner, [p], r in "wallet_pool", on: r.wallet_id in ^wallet_ids and p.id == r.pool_id)
    |> order_by([_, r], r.priority)
    |> select([p, r], {r.wallet_id, merge(p, %{priority: priority})})
    |> Repo.all()
  end
end

iex> WalletHelper.preload_pools([99])
[
  {99,
     %Pool{
       id: 32,
       currency: "BTC",
       port: 25,
       url: "stratum+tcp://stratum.example.com",
       priority: "first"
    }
  },
  {99,
     %Pool{
       id: 33,
       currency: "BTC",
       port: 25,
       url: "stratum+tcp://stratum.example2.com",
       priority: "second"
    }
  },
]
```

That's it, now we are able to compose `wallet` queries while preloading pools, and the `priority` of the pool for each `wallet.
