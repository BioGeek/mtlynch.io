---
title: Why Good Developers Write Bad Unit Tests
layout: single
author_profile: true
read_time: true
comments: true
share: true
related: true
sidebar:
  nav: main
classes: wide
header:
  teaser: images/good-developers-bad-tests/cover.jpg
  og_image: images/good-developers-bad-tests/cover.jpg
---

Congratulations! You've finally written so many lines of code that you can afford your very own beach house. Money is no object, so you hire Peter Keating, world-famous architect. He gained his reputation with skyscrapers, but he assures you that he has brilliant plans for your beachfront property.

Months later, you arrive at the grand unveiling. Your new home is an imposing five-story behemoth of steel, concrete, and reflective glass. You track sand onto the marble floor as you pass through the revolving doors. You're greeted by a reception desk which sits in front of an elevator bank. Upstairs, your master bedroom and three guest rooms are just four adjoining office cubicles.

{% include image.html file="cover.jpg" alt="Architect presenting skyscraper on the beach" max_width="800px" img_link=true %}

Peter Keating, expert architect, can't understand why you're disappointed. "I followed **all** the best practices," he tells you, defensively. The walls are three feet thick because structural integrity is important to a building. Therefore, your home is *better* than the breezy, light-filled homes neighboring it. You may not have large, oceanside windows, but Keating tells you that such windows are not best practice &mdash; they reduce energy efficiency and distract office workers.

Too often, software developers approach unit testing with the same flawed thinking. They mechanically apply all the "rules" they learned in production code without examining whether they're appropriate for tests. As a result, they build skyscrapers at the beach.

# Test code is not like other code

Production code is all about abstractions. Good production code hides complexity in well-scoped functions and beautiful class hierarchies. It allows the reader to navigate a large system with ease, diving down into the details or jumping up to a higher level of abstraction.

Test code is a different beast. Every layer of abstraction in a test makes it harder to understand. Tests are a diagnostic tool, so they should be as simple and obvious as possible.

**Good production code is well-factored; good test code is *obvious*.**
{: .notice--info}

{% include image.html file="dane-deaner-272363-unsplash-cropped.jpg" alt="Image of a ruler" img_link=true max_width="325px" class="align-right" link_url="https://unsplash.com/photos/JNpmCYZID68" %}

Think of a ruler. It's existed unchanged for hundreds of years because it's simple and easy to interpret. Suppose I invented a new ruler that measured in "abstract ruler units." Users could convert from "ruler units" to inches or centimeters with a conversion chart.

If I offered such a ruler to a carpenter, they'd smack me in the face with it. When they use a ruler, they want the simplest answer and minimal layers of abstraction.

Good test code is the same way. It should minimize the reader's cognitive load.

# A good developer's bad test

I often see otherwise talented developers write tests that look like the following:

```python
def test_initial_score(self):
  initial_score = self.account_manager.get_score(username='joe123')
  self.assertEqual(150.0, initial_score)
```

What does that test do? It retrieves a "score" for a user with the name `joe123` and verifies that the user's score is 150. At this point, you should have the following questions:

1. Where did the `joe123` account come from?
1. Why do I expect `joe123`'s score to be 150?

Perhaps the answers are in the `setUp` method, which the test framework calls before executing each test function:

```python
def setUp(self):
  database = MockDatabase()
  database.add_row({
      'username': 'joe123',
      'score': 150.0
    })
  self.account_manager = AccountManager(database)
```

Okay, the `setUp` method created the `joe123` account with a score of 150, which explains why `test_initial_score` expected those values. Now, all is well with the world, right?

No, this is a **bad test**.

# Keep the reader in your test function

When you write a test, think about the next developer who will see the test break. They don't want to read your entire test suite, and they certainly don't want to read an inheritance tree of test classes.

If a test breaks, the reader should be able to diagnose the problem quickly by reading the body of the test function in a straight line from top to bottom. If they have to jump out of the test to read ancillary code, the test has not done its job.

With this in mind, here's a rewrite of the test from the previous section:

```python
def test_initial_score(self):
  database = MockDatabase()
  database.add_row({
      'username': 'joe123',
      'score': 150.0
    })
  account_manager = AccountManager(database)

  initial_score = account_manager.get_score(username='joe123')

  self.assertEqual(150.0, initial_score)
```

All I did was inline the code from the `setUp` method, but it made a world of difference. Now, everything the reader needs is right there in the test. It also has a clear structure of [arrange, act, assert](http://wiki.c2.com/?ArrangeActAssert), so the structure itself aids in the reader's comprehension.

**The reader should understand a test function without reading any code outside the function body.**
{: .notice--info}

# Dare to violate DRY

Inlining the setup code is all well and good for a single test. But what happens if I have many tests? Won't I duplicate that code every time? Prepare yourself, because you'll likely find my answer shocking: I'm about to advocate [copy/paste programming](https://en.wikipedia.org/wiki/Copy_and_paste_programming).

Here's another test of the same class:

```python
def test_increase_score(self):
  database = MockDatabase()                  # <
  database.add_row({                         # <
      'username': 'joe123',                  # <--- Copy/pasted from
      'score': 150.0                         # <--- previous test
    })                                       # <
  account_manager = AccountManager(database) # <

  account_manager.adjust_score(username='joe123',
                         adjustment=25.0)

  self.assertEqual(175.0,
             account_manager.get_score(username='joe123'))
```

For strict adherents to the [principle of DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) ("don't repeat yourself"), the above code is horrifying. I'm blatantly repeating myself. Worse, I'm arguing that my DRY-violating tests are **better** than tests that are free of repeated code. How can this be?

If you can achieve clear tests without repeating code, that's ideal, but remember that nonredundant code is the means, not the goal. Tests should strive to be as clear and obvious as possible. Instead of blindly obeying DRY, think about what will make the next developer understand the problem quickly when this test fails.

**The reader should understand a test function without reading any code outside the function body.**
{: .notice--info}

# Think twice before adding helper methods

Maybe you can live with copy/pasting six lines in every test, but what if `AccountManager` required more setup code?

```python
def test_increase_score(self):
  # vvvvvvvvvvvvvvvvvvvvv Beginning of boilerplate code vvvvvvvvvvvvvvvvvvvvv
  user_database = MockDatabase()
  user_database.add_row({
      'username': 'joe123',
      'score': 150.0
    })
  privilege_database = MockDatabase()
  privilege_database.add_row({
      'privilege': 'upvote',
      'minimum_score': 200.0
    })
  privilege_manager = PrivilegeManager(privilege_database)
  url_downloader = UrlDownloader()
  account_manager = AccountManager(user_database,
                                   privilege_manager,
                                   url_downloader)
  # ^^^^^^^^^^^^^^^^^^^^^ End of boilerplate code ^^^^^^^^^^^^^^^^^^^^^^^^^^^

  account_manager.adjust_score(username='joe123',
                         adjustment=25.0)

  self.assertEqual(175.0,
             account_manager.get_score(username='joe123'))
```

That's 15 lines just to get an instance of `AccountManager` and begin testing it. At that level, there's so much boilerplate that it distracts from the behavior you're testing.

Your natural inclination might be to bury all the uninteresting code in test helper methods, but you should first ask a more vital question: why is the system so difficult to test?

If your system requires excessive boilerplate code, that may indicate a flaw in your design. For example, take another look at how I created the `AccountManager` object in the test above:

```python
account_manager = AccountManager(user_database,
                                 privilege_manager,
                                 url_downloader)
```

There are design smells here. It accesses the `user_database` directly but its next parameter is `privilege_manager`, a wrapper for `privilege_database`. Why is `AccountManager` operating on two different layers of abstraction? And what is it doing with a "URL downloader?" That's certain seems conceptually distant from its other two parameters.

In this case, refactoring `AccountManager` solves the root problem whereas helper methods would simply bury the symptoms. A more cohesive interface makes the class easier to use in both test and production code.

**When there's repetitive test code, eliminate it with clean interfaces before adding test helper methods.**
{: .notice--info}

# If you need helper methods, write them responsibly

You don't always have the freedom to tear apart a production class for testability. Sometimes, helper methods are your only choice, so when you need them, write them well.

A good helper method supports the principle of "keep the reader in your test function." It's okay to extract boilerplate code into a helper as long as the reader can ignore that code without losing understanding of the test.

Specifically, helper methods should **not**:

* bury critical values
* interact with the object under test

Here's an example of a helper method that violates these guidelines:

```python
def add_dummy_account(self): # <- Helper method
  dummy_account = Account(username='joe123',
                          name='Joe Bloggs',
                          email='joe123@example.com',
                          score=150.0)
  self.account_manager.add_account(dummy_account) # BAD: Helper method hides an interesting call

def test_increase_score(self):
  self.account_manager = AccountManager()
  self.add_dummy_account()

  account_manager.adjust_score(username='joe123',
                               adjustment=25.0)

  self.assertEqual(175.0, # BAD: Relies on value set in helper method
                   account_manager.get_score(username='joe123'))
```

The reader can't understand why the final score should be `175.0` unless they search out the `150.0` hidden in the helper method. The helper also obscures `account_manager`'s behavior by hiding a call to `add_account` instead of keeping all interactions in the test function itself.

Here's a rewrite that addresses those issues:

```python
def make_dummy_account(self, username, score):
  return Account(username=username,
                 name='Dummy User',         # <- OK: Buries values but they're
                 email='dummy@example.com', # <-     irrelevant to the test
                 score=score)

def test_increase_score(self):
  account_manager = AccountManager()
  account_manager.add_account(
    make_dummy_account(
      username='joe123',  # <- GOOD: Relevant values stay
      score=150.0))       # <-       in the test

  account_manager.adjust_score(username='joe123',
                               adjustment=25.0)

  self.assertEqual(175.0,
                   account_manager.get_score(username='joe123'))
```

It still buries values in the helper method, but they're not relevant to the test, so the reader doesn't have to care about them. It also pulls the `add_account` call back into the test so that the reader can trivially trace everything that happens to `account_manager`.

# Go crazy with long test names

Which of the following function names would you prefer to see in production code?

* `userExistsAndTheirAccountIsInGoodStandingWithAllBillsPaid`
* `isAccountActive`

The first conveys more information, but comes with the burden of a 57-character name. Most developers are willing to sacrifice a bit of precision in favor of for a concise, almost-as-good name like `isAccountActive` (except for Java developers, for whom both names are offensively terse).

For test functions, there's a key factor that changes the equation: you never write *calls* to test functions. A developer types out a test name exactly once &ndash; in the function signature. Given this, brevity still matters, but it matters less than in production code.

Whenever a test breaks, the test name is the first thing you see, so it should convey as much information as possible. For example, consider this production class:

```c++
class Tokenizer {
 public:
  Tokenizer(std::unique_ptr<TextStream> stream);
  std::unique_ptr<Token> NextToken();
 private:
  std::unique_ptr<TextStream> stream_;
};
```

Suppose you ran your test suite and this line appeared in the output:

```text
[  FAILED  ] TokenizerTests.TestNextToken (6 ms)
```

Would you know what caused the test to fail? Probably not.

A failure in `TestNextToken` tells you that you screwed up the `NextToken()` method, but that's meaningless in a class with a single public method. To diagnose the failure, you'd  have to read the test itself to understand the problem.

Instead, what if you saw this:

```text
[  FAILED  ] TokenizerTests.ReturnsNullptrWhenStreamIsEmpty (6 ms)
```

A function called `ReturnsNullptrWhenStreamIsEmpty` would feel overly verbose in other contexts, but it's a good test name. It tells the reader exactly what behavior it verifies. A developer could fix this break without ever reading the test's implementation. That's the mark of a good test name.

**A good test name is so descriptive that a developer can diagnose failures of that test from the name alone.**
{: .notice--info}

# Embrace magic numbers

"Don't use magic numbers."

It's the "don't talk to strangers" of the programming world. It becomes so ingrained in many talented developers that they never consider when a magic number might improve their code. But, here's a secret: magic numbers make unit tests better.

As a refresher, a "magic number" is a numeric value or string that appears in code without information about what it represents. Here's an example:

```python
calculate_pay(80) # <-- magic number
```

Programmers agree that magic numbers in production code are A Very Bad Thing, so they replace them with named constants like this:

```python
HOURS_PER_WEEK = 40
WEEKS_PER_PAY_PERIOD = 2
calculate_pay(hours=HOURS_PER_WEEK * WEEKS_PER_PAY_PERIOD)
```

Unfortunately, there's a misconception that magic numbers also weaken *test* code, but the opposite is true.

Consider the following test:

```python
def test_add_hours(self):
  TEST_STARTING_HOURS = 72.0
  TEST_HOURS_INCREASE = 8.0
  hours_tracker = BillableHoursTracker(initial_hours=TEST_STARTING_HOURS)
  hours_tracker.add_hours(TEST_HOURS_INCREASE)
  expected_billable_hours = TEST_STARTING_HOURS + TEST_HOURS_INCREASE
  self.assertEqual(expected_billable_hours, hours_tracker.billable_hours())
```

If you believe magic numbers are universally evil, the above test looks correct to you. `72.0` and `8.0` have named constants, so nobody can accuse the test of magic numbers. But for a moment, suspend your religious beliefs and indulge in the forbidden fruit of magic numbers:

```python
def test_add_hours(self):
  hours_tracker = BillableHoursTracker(initial_hours=72.0)
  hours_tracker.add_hours(8.0)
  self.assertEqual(80.0, hours_tracker.billable_hours())
```

It's simpler, with only half as many lines of code. And it's more obvious &mdash; the reader doesn't have to jump around to track names of constants.

Magic numbers made this test better, just as they make all tests better. If you find yourself yearning to define a constant in your test, it likely indicates a flaw in the test itself, such as [abuse of helper methods](#if-you-need-helper-methods-write-them-responsibly).

**Avoid creating named constants in test code. Use magic numbers instead.**
{: .notice--info}

# Conclusion

Software development is engineering. A skilled engineer does more than just memorize a list of rules and apply them universally. Engineering requires one to understand the fundamental principles of their craft and to weigh the benefits and drawbacks of different decisions.

To write excellent tests, a developer must recognize how the goals of test code differ from those of production code and adjust their engineering decisions accordingly. Most importantly, tests should maximize simplicity and minimize abstraction.