# SpamAssassin and Finnish language

Just wanted to share some useful spam preventing rules matching Finnish language.

You can add and maintain your SpamAssassin rules in directory:

```shell
~$ cd /usr/share/spamassassin
```

One practice is to edit existing rules and the other is to add a separate custom rule file:

```shell
/usr/share/spamassassin$ vi 90_custom.cf
```

Content of the rule file leads to SpamAssassin documentation https://wiki.apache.org/spamassassin/WritingRules

I'm currently using following content:

```plain
require_version @@VERSION@@

header LOAN_FINNISH_1          Subject =~ /\bLaina(a|hakemus|tarjouksia|tarjous|si|n)\b/i
body LOAN_FINNISH_1            /\bLaina(a|hakemus|tarjouksia|tarjous|si|n)\b/i
describe LOAN_FINNISH_1        Offers loan in Finnish language

header GIFT_FINNISH_1          Subject =~ /\bLahjakort(timme|ti|it|teja|in|tisi)\b/i
body GIFT_FINNISH_1            /\bLahjakort(timme|ti|it|teja|in|tisi)\b/i
describe GIFT_FINNISH_1        Offers gifts in Finnish language

if !(! plugin (Mail::SpamAssassin::Plugin::URIDetail))
  uri_detail URL_UNSUBSCRIBE_1   text =~ /\bunsubscribe\b/i
  describe URL_UNSUBSCRIBE_1     Contains URI with text unsubscribe
endif

score LOAN_FINNISH_1 3 # n=0 n=1 n=2 n=3
score GIFT_FINNISH_1 3 # n=0 n=1 n=2 n=3
score URL_UNSUBSCRIBE_1 3 # n=0 n=1 n=2 n=3
```

It is a good practice to test rule syntax after adding or editing ruleset:

```plain
~$ /usr/bin/spamassassin --lint
```

No output should be generated.
