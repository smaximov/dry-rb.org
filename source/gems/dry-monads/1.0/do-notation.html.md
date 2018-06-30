---
title: Do Notation
layout: gem-single
name: dry-monads
---


Composing several monadic values can become tedious because you need to pass around unwrapped values in lambdas (aka blocks). Haskell was one of the first languages faced this problem. To work around it Haskell has a special syntax for combining monadic operations called the "do notation". If you're familiar with Scala it has `for`-comprehensions for a similar purpose. It is not possible to implement `do` in Ruby but it is possible to emulate it to some extent, i.e. achieve comparable usefulness.

What `Do` does is passing an unwrapping block to certain methods. The block tries to extract the underlying value from a monadic object and either short-circuits the execution (in case of a failure) or returns the unwrapped value back.

See the following example written using `bind` and `fmap`:

```ruby
class CreateAccount
  include Dry::Monads::Result::Mixin

  def call(params)
    validate(params).bind { |values|
      create_account(values[:account]).bind { |account|
        create_owner(account, values[:owner]).fmap { |owner|
          [account, owner]
        }
      }
    }
  end

  def validate(params)
    # returns Success(values) or Failure(:invalid_data)
  end

  def create_account(account_values)
    # returns Success(account) or Failure(:account_not_created)
  end

  def create_owner(account, owner_values)
    # returns Success(owner) or Failure(:owner_not_created)
  end
end
```

The more monadic steps you need to combine the harder it becomes, not to mention how difficult it can be to refactor code written in such way.

Embrace `Do`:

```ruby
require 'dry/monads/result'
require 'dry/monads/do'

class CreateAccount
  include Dry::Monads::Result::Mixin
  include Dry::Monads::Do.for(:call)

  def call(params)
    values = yield validate(params)
    account = yield create_account(values[:account])
    owner = yield create_owner(account, values[:owner])

    Success([account, user])
  end

  def validate(params)
    # returns Success(values) or Failure(:invalid_data)
  end

  def create_account(account_values)
    # returns Success(account) or Failure(:account_not_created)
  end

  def create_owner(account, owner_values)
    # returns Success(owner) or Failure(:owner_not_created)
  end
end
```

Both snippets do the same thing yet the second one is a lot easier to deal with. All what `Do` does here is prepending `CreateAccount` with a module which passes a block to `CreateAccount#call`. That simple.

### Transaction safety

Under the hood, `Do` uses `throw`/`catch` to halt unsuccessful operations, this can be slower if you are dealing with unsuccessful paths a lot, but usually, this is not an issue. Check out [this article](https://www.morozov.is/2018/05/27/do-notation-ruby.html) for actual benchmarks.

One particular reason to use exceptions is the ability to make code transaction-friendly. In the example above, this piece of code is not atomic:

```ruby
account = yield create_account(values[:account])
owner = yield create_owner(account, values[:owner])

Success([account, user])
```

What if `create_account` succeeds and `create_owner` fails? This will leave your database in an inconsistent state. Let's wrap it with a transaction block:

```ruby
repo.transaction do
  account = yield create_account(values[:account])
  owner = yield create_owner(account, values[:owner])

  Success([account, user])
end
```

Since `yield` internally uses exceptions to control the flow, the exception will be detected by the `transaction` call and the whole operation will be rolled back. No more garbage in your database, yay!

### Limitations

`Do` only works with single-value monads, i.e. most of them. At the moment, there is no way to make it work with `List`, though.

### Adding batteries

The `Do::All` module takes one step ahead, it tracks all new methods defined in the class and passes a block to every one of them. However, if you pass a block yourself then it takes precedence. This way, in most cases you can use `Do::All` instead of listing methods with `Do.for(...)`:

```ruby
require 'dry/monads/result'
require 'dry/monads/do/all'

class CreateAccount
  include Dry::Monads::Result::Mixin
  include Dry::Monads::Do::All

  def call(account_params, owner_params)
    repo.transaction do
      account = yield create_account(account_params)
      owner = yield create_owner(account, owner_params)

      Success([account, owner])
    end
  end

  def create_account(params)
    values = yield validate_account(params)
    account = repo.create_account(values)

    Success(account)
  end

  def create_owner(account, params)
    values = yield validate_owner(params)
    owner = repo.create_owner(account, values)

    Success(owner)
  end

  def validate_account(params)
    # returns Success/Failure
  end

  def validate_owner(params)
    # returns Success/Failure
  end
end
```
