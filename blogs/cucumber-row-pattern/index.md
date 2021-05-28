# Cucumber Row Pattern

_Pattern for data loading and verification in Cucumber acceptance tests._

_cucumber, acceptance tests, data loading, data verification, data tables_

30/03/2017

![Cucumber Row Pattern](greenhouse-2139526_1920.jpg)

Many projects at Black Pepper benefit from [behaviour-driven development](https://en.wikipedia.org/wiki/Behavior-driven_development) (BDD) which allows acceptance tests to be understood by business stakeholders and yet still be executable by machines. Our go-to tool for this is typically [Cucumber](https://cucumber.io/) as its specifications are very readable and it enjoys support across [many platforms](http://specflow.org/).

Getting started with Cucumber is straightforward enough but certain design decisions arise once the complexity of a system increases. One that has routinely occurred across our projects is how best to implement step definitions that load and verify data in an application. I'm going to dub this the _Cucumber Row Pattern_ and attempt to detail it here.

## Scenario

Let's consider a simple banking application. Our bank will hold multiple accounts each consisting of a name and a balance. A Cucumber scenario to test that we can add accounts might look like this:

```gherkin
Scenario: Accounts can be created
  Given the system has no accounts
  When the user adds the following accounts
    | Name       | Balance |
    | Chip Smith | 100.00  |
    | Randy Horn | 200.00  |
    | Zane High  | 300.00  |
  Then the system has the following accounts
    | Name       | Balance |
    | Chip Smith | 100.00  |
    | Randy Horn | 200.00  |
    | Zane High  | 300.00  |
```

The interesting steps here are the latter two which use [data tables](https://cucumber.io/docs/reference#data-tables). The first of which puts data into the system and the second asserts that the correct data is present in the system. This is a recurring requirement for data within an application and one that this pattern tries to standardise.

## Data loading steps

Let's first consider the data loading step. It's tempting to convert the data table directly into domain objects for the application to consume. The primary drawback of this approach is that the acceptance tests become tightly coupled to the implementation, leaving little room for manoeuvre as they inevitably diverge.

To remedy this we can introduce a row class that in turn produces the domain model. We can then use this to write our step definition:

```java
When("^the user adds the following accounts$", (DataTable accounts) -> {
    accounts.asList(AccountRow.class)
        .stream()
        .map(AccountRow::toModel)
        .forEach(bank::addAccount);
});
```

(Note that we're using the [Java 8 lambda style](https://cucumber.io/docs/reference/jvm#lambda-expressions-java-8) which unfortunately means that we cannot inject a `List<AccountRow>` as [Cucumber cannot infer generic types in lambdas](https://github.com/cucumber/cucumber-jvm/issues/937) yet.)

The row class is a simple POJO with a `toModel()` method that produces `Account` domain objects:

```java
public class AccountRow {

    private String name;

    private BigDecimal balance;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public BigDecimal getBalance() {
        return balance;
    }

    public void setBalance(BigDecimal balance) {
        this.balance = balance;
    }

    public Account toModel() {
        return new Account(name, balance);
    }
}
```

## Data verification steps

Our second requirement is to be able to verify data in the system. In our scenario above, the verification step used similar shaped data to the loading step, so it makes sense to reuse our row class for this purpose.

Let's add a `toMatcher()` method to our row class that produces a [Hamcrest](http://hamcrest.org/JavaHamcrest/) matcher:

```java
public Matcher<Account> toMatcher() {
    return allOf(
        hasProperty("name", equalTo(name)),
        hasProperty("balance", equalTo(balance))
    );
}
```

We can then use this to easily implement our verification step definition:

```java
Then("^the system has the following accounts$", (DataTable accounts) -> {
    assertThat(bank.getAccounts(), containsInAnyOrder(accounts.asList(AccountRow.class)
        .stream()
        .map(AccountRow::toMatcher)
        .collect(toList())
    ));
});
```

## Recap

Before we go any further it's worth summarising the pattern so far:

```
                            DataTable
                                |
                                | asList()
                                |
                                V
                               Row
                                |
                        ________|________
                       |                 |
             toModel() |                 | toMatcher()
                       |                 |
                       V                 V
                     Model            Matcher
```

We convert Cucumber data tables into row objects, from which we can either create domain models to load data into the system, or create matchers to verify data in the system.

## Default values

So far so good. Now, a new requirement comes in to support current and savings accounts. No problem, we can achieve this by adding a mandatory _type_ to our account model, but isn't this going to propagate through to all our scenario data tables? I'd rather not have to update every one with a new column when it's immaterial to the test.

What we need is to support missing columns by setting the corresponding model property to a default value. That way our scenarios stay uncluttered and our domain model invariants are happy. Let's default all properties of an account:

```java
public Account toModel() {
    return new Account(
        Optional.ofNullable(name).orElse("Unnamed"),
        Optional.ofNullable(type).orElse(CURRENT),
        Optional.ofNullable(balance).orElse(ZERO)
    );
}
```

## Sparse matchers

Continuing to add support for account types to our matcher poses a problem. Because the original scenario didn't specify an account type column then it arrives as `null` in the row object, causing the matcher to assert this when we really want to ignore it.

The behaviour we'd like is to only verify model attributes when they are explicitly specified as columns. We can produce sparse matchers such as these as follows:

```java
public Matcher<Account> toMatcher() {
    return allOf(
        Stream.of(
            Optional.ofNullable(name).map(value -> hasProperty("name", equalTo(value))),
            Optional.ofNullable(type).map(value -> hasProperty("type", equalTo(value))),
            Optional.ofNullable(balance).map(value -> hasProperty("balance", equalTo(value)))
        )
        .filter(Optional::isPresent)
        .map(Optional::get)
        .collect(toList())
    );
}
```

## Value expressions

Although we should always strive to write scenarios in plain English, situations do arise where there's a need for a primitive expression language. A common problem is asserting time sensitive data. For example, consider a new requirement to record the date that accounts are opened. How would we specify the expected date in a data table when time keeps marching on?

One solution is to use a placeholder for the current date:

```gherkin
Scenario: Accounts have an opened date
  Given the system has no accounts
  When the user adds the following accounts
    | Name       |
    | Chip Smith |
  Then the system has the following accounts
    | Name       | Opened  |
    | Chip Smith | [today] |
```

Here we introduce a simple expression of `[today]` that evaluates to the current date. Where does this get evaluated? For expressions that equate to a single non-null value we introduce an `evaluate()` method on the row class and invoke it from our step definitions:

```java
When("^the user adds the following accounts$", (DataTable accounts) -> {
    accounts.asList(AccountRow.class)
        .stream()
        .map(AccountRow::evaluate)
        .map(AccountRow::toModel)
        .forEach(bank::addAccount);
});

Then("^the system has the following accounts$", (DataTable accounts) -> {
    assertThat(bank.getAccounts(), containsInAnyOrder(accounts.asList(AccountRow.class)
        .stream()
        .map(AccountRow::evaluate)
        .map(AccountRow::toMatcher)
        .collect(toList())
    ));
});
```

This new method simply returns a new row with any columns that support expressions evaluated:

```java
public AccountRow evaluate() {
    AccountRow row = new AccountRow();
    row.setName(name);
    row.setType(type);
    row.setBalance(balance);
    row.setOpened(evaluateValue(opened));
    return row;
}
```

The code to actually evaluate an expression isn't that important here and can be simply performed by static methods. Let's introduce an `Expressions` class for this:

```java
public final class Expressions {

    public static String evaluateValue(String expression) {
        return Optional.ofNullable(expression)
            .map(value -> value.replace("[today]", LocalDate.now().toString()))
            .orElse(null);
    }
}
```

Note that the opened date in the row class is of type `String` rather than `LocalDate` to allow it to hold expressions such as `[today]`. After evaluation these values are then parsed in the row class as follows:

```java
public Account toModel() {
    return new Account(
        ...
        Optional.ofNullable(opened).map(LocalDate::parse).orElse(MIN)
    );
}

public Matcher<Account> toMatcher() {
    return allOf(
        Stream.of(
            ...
            Optional.ofNullable(opened).map(value -> hasProperty("opened",
                equalTo(LocalDate.parse(value))
            ))
        ...
    );
}
```

## Matcher expressions

Another type of expression frequently encountered is one that resolves to a range of values. For example, say we wanted to assert that an account's opened date was in the past, how would we achieve that?

Since this expression cannot be evaluated to a single value then we need to handle it outside of `evaluate()`. Instead we can process it when building the matcher:

```java
public Matcher<Account> toMatcher() {
    return allOf(
        Stream.of(
            ...
            Optional.ofNullable(opened).map(value -> hasProperty("opened",
                evaluateMatcher(value).orElseGet(() -> equalTo(LocalDate.parse(value)))
            ))
        )
        ...
    );
}
```

This attempts to evaluate the property as a matcher expression, falling back on `equalTo()` if it's a regular value. Note the lazy `orElseGet()` to prevent illegally parsing an expression as a date. Again, we've delegated the parsing to another evaluation method:

```java
public final class Expressions {
    ...
    public static Optional<Matcher<?>> evaluateMatcher(String expression) {
        return Optional.ofNullable("[past]".equals(expression) ? lessThan(LocalDate.now()) : null);
    }
}
```

We can now use the expression `[past]` to match any date in the past.

## Null expressions

When discussing expressions above we were careful to limit them to those that evaluate to a non-null value. Why was this? The problem is that if an expression evaluates to null then, by design, it is defaulted in the model and ignored in the matcher. So what if you really want to set a property to null or assert that it is indeed null?

The solution is to ignore null expressions in `evaluate()` and instead handle them when building the model and matcher. For example, to support null account names in data loading steps:

```java
public Account toModel() {
    return new Account(
        evaluateNullValue(Optional.ofNullable(name).orElse("Unnamed")),
        ...
    );
}
```

Where the new evaluation method solely parses `[null]` expressions:

```java
public final class Expressions {
    ...
    public static String evaluateNullValue(String expression) {
        return "[null]".equals(expression) ? null : expression;
    }
}
```

Supporting null expressions in data verification steps is slightly easier as they just become another type of matcher expression:

```java
public static Optional<Matcher<?>> evaluateMatcher(String expression) {
    Matcher<?> matcher = null;

    if ("[past]".equals(expression)) {
        matcher = lessThan(LocalDate.now());
    }
    else if ("[null]".equals(expression)) {
        matcher = nullValue();
    }

    return Optional.ofNullable(matcher);
}
```

## Summary

We've covered quite a lot of ground here so it's worth reiterating the key parts of the pattern:

```
                            DataTable
                                |
                                | asList()
                                |
                                V
                               Row
                                |
                                | evaluate()
                                | - evaluate non-null value expressions
                                V
                               Row
                                |
                        ________|________
                       |                 |
             toModel() |                 | toMatcher()
- apply default values |                 | - ignore null values
- evaluate null value  |                 | - evaluate matcher expressions
  expressions          |                 |
                       V                 V
                     Model            Matcher
```

Row classes act as an intermediary between Cucumber data tables and the domain. They can produce models and matchers to load and verify data in the system by using `toModel()` and `toMatcher()` respectively.

Data table columns can be omitted to use default values in the model or ignore properties during assertion by using `Optional` in the row class.

Single value expressions are evaluated across the row class in `evaluate()`, unless they equate to `null` in which case they must be processed when building models. Whereas multi-value expressions are always handled when building matchers due to their indeterminate nature.

Remember that not all these features are required for all types of data in a system, so I would suggest a pick-and-mix approach when implementing this pattern. For the same reason, and for the sake of simplicity, I'm not yet convinced by the need to wrap this up under the guise of another API.

I appreciate that lots of code snippets can be hard to follow, so be sure to head over to [cucumber-row-demo](https://github.com/markhobson/cucumber-row-demo) to see the pattern in action.
