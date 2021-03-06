# ruy

Ruy is a library for defining a set of conditions and evaluating them against a context.

``` ruby
# discount_day.rb
gifter = Ruy::RuleSet.new

gifter.eq :friday, :day_of_week

gifter.outcome 8 do
  greater_than_or_equal 300, :amount
end

gifter.outcome 7 do
  greater_than_or_equal 100, :amount
end

gifter.outcome 3

gifter.fallback 0
```

RuleSets are evaluated against a context (the `Hash` being passed to `#call`) and return the first outcome that matches.

``` ruby
gifter.call(day_of_week: :friday, amount: 314)

# => 8
```

``` ruby
gifter.call(day_of_week: :friday, amount: 256)

# => 7
```

If no outcome matches, the default one is returned.
``` ruby
gifter.call(day_of_week: :friday, amount: 99)

# => 3
```

If conditions are not met, the fallback value is returned.
``` ruby
gifter.call(day_of_week: :monday, amount: 124)

# => 0
```

## Key concepts

Ruy at its core is about evaluating a set of conditions against a context in order to return a result.

### Conditions

A condition evaluates the state of the context.

Available conditions:

 - all *All of the nested conditions must suffice*
 - any *At least one of its nested conditions must suffice*
 - assert *A context value must be truish*
 - between *Evaluates that a context value must belong to a specified range*
 - cond *At least one slice of two nested conditions must suffice*
 - day_of_week *Evaluates that a Date/DateTime/Time weekday is matched*
 - eq *Tests a context value for equality*
 - except *Evaluates that a context value is not equal to a specified value*
 - greater_than *Tests that context value is greater than something*
 - greater_than_or_equal *Tests that context value is greater than or equal to something*
 - in *A context value must belong to a specified list of values*
 - in_cyclic_order *TBD*
 - include *The context attribute must include a specified value*
 - less_than_or_equal *Tests that context value is less than or equal to something*
 - less_than *Tests that context value is less than something*

Conditions can be nested. In such case, for the nesting condition to be met, the nested conditions must
also be met.

``` ruby
between 0, 1_000, :amount do
  eq :friday, :day_of_week
end
```

is equivalent to:

``` ruby
all do
  between 0, 1_000, :amount
  eq :friday, :day_of_week
end
```

### Rulsets

A ruleset is a set of conditions that must suffice and returns a value resulting from either an
outcome or a fallback.

### Contexts

A context is a `Hash` from which values are fetched in order to evaluate a ruleset.

### Lazy values

Rulesets can define lazy values. The context must provide a proc which is evaluted only once the first time the value is needed. The result returned by the proc is memoized and used to evaluate subsequent conditions.


``` ruby
# premium_discount_day.rb
gifter = Ruy::RuleSet.new

gifter.let :amounts_average # an expensive calculation

gifter.eq :friday, :week_of_day

gifter.greater_than_or_equal 10_000, :amounts_average

gifter.outcome true
```

``` ruby
gifter.call(day_of_week: :friday, amounts_average: -> { Stats::Amounts.compute_average })
```
### Outcomes

An outcome is the result of a successful ruleset evaluation. An outcome can also have nested
conditions, in such case, if the conditions meet, the outcome value is returned.

A RuleSet can have multiple outcomes, the first matching one is returned.

### Time Zone awareness

When it comes to matching times in different time zones, Ruy is bundled with a built in `tz` block that will enable specific matchers to make time zone-aware comparisons.

```ruby
ruleset = Ruy::RuleSet.new

ruleset.tz 'America/New_York' do
  eq '2015-01-01T00:00:00', :timestamp
end

ruleset.outcome 'Happy New Year, NYC!'
```

For example, if the timestamp provided in the context is a Ruby Time object in UTC (zero offset from UTC), `eq` as child of a `tz` block will take the time zone passed as argument to the block (`America/New_York`) to calculate the current offset and make the comparison.

String time patterns follow the Ruy's well-formed time pattern structure as follows:

`YYYY-MM-DDTHH:MM:SS[z<IANA Time Zone Database identifier>]`

Where the time zone identifier is optional, but if you specify it, will take precedence over the block's identifier. In case you don't specify it, Ruy will get the time zone from the `tz` block's argument. If neither the block nor the pettern specify it, UTC will be used.

#### Days of week matcher

Inside any `tz` block, there's a matcher to look for a specific day of the week in the time zone of the block.

```ruby
ruleset = Ruy::RuleSet.new

ruleset.any do
  tz 'America/New_York' do
      day_of_week :saturday, :timestamp
  end

  tz 'America/New_York' do
      day_of_week 0, :timestamp # Sunday
  end
end

ruleset.outcome 'Have a nice weekend, NYC!'
```

This matcher supports both the `Symbol` and number syntax in the range `(0..6)` starting on Sunday.

The day of week matcher will try to parse timestamps using the ISO8601 format unless the context passes a Time object.

#### Nested blocks support

You cannot use matchers inside nested blocks in a `tz` block expecting them to work as if they were immediate children of `tz`.

A possible workaround for this is to use `tz` blocks inside the nested block in question:

```ruby
ruleset = Ruy::RuleSet.new
any do
  tz 'America/New_York' { eq '2015-01-01T00:00:00', :timestamp }
  tz 'America/New_York' { eq '2015-01-01T02:00:00zUTC', :timestamp }
end

ruleset.outcome 'Happy New Year, NYC!'
```

Support for time zone awareness in nested blocks inside `tz` blocks is planned. This workaround could stop working in future versions; use it at your own risk.

Ruy depends on [TZInfo](http://tzinfo.github.io/ "TZ Info website") to calculate offsets using IANA's Time Zone Database. Check their website for information about time zone identifiers.
