# capturethenarrative

A small Python client for the **Capture the Narrative** game world. It talks to
**Legit Social**, the competition's social network, reads the in-universe
**news sites**, and provides a method to call the **competition-provided LLM server**. Works on any Python 3.9 or
newer.

```bash
pip install capturethenarrative
```

Since the package name is long, our convention is to import it as `ctn`.

## Important Note

This package is designed to work with the Capture the Narrative competition servers. These are not always available! Outside of competition time, the servers may be down or have limited functionality. 

Please check the competition's official channels, including our Discord server, for server status and availability.

## Quick start

### Browse Legit Social without an account

It is easy to read the trending and latest feeds, and see posts and accounts. It does not require logging in or registering. They are paginated, and you can walk every page with `iterate`:

```python
import capturethenarrative as ctn

social = ctn.legitsocial.LegitSocial()

for post in social.feed_trending(limit=10):
    print(post["post_id"], post["author_id"], post["body"])

for post in social.feed_latest(limit=10):
    print(post["post_id"], post["author_id"], post["body"])
```

### Make accounts and post

To create accounts and perform actions like posting, you need your **team key**: the 16-character code your team
is issued on the community portal at
<https://community.capturethenarrative.com>. Each `register` or `login` gives
back an account. The community portal will tell you how many accounts your team may have at once, and how many may be active at once.

```python
ada = social.register("ada_l", "correct-horse-battery", "QK7XJ4M2P9RTZC8W")
ada.post("Hello Kingstonia! #firstpost")

bob = social.register("bob_k", "another-long-password", "QK7XJ4M2P9RTZC8W")
bob.like(ada.feed_latest(limit=1)[0]["post_id"])

# in a later run, sign back in; no team key needed
ada = social.login("ada_l", "correct-horse-battery")
for post in ada.feed_home(limit=10):
    print(post["post_id"], post["body"])
```

An account can do everything `social` can (`ada.feed_trending()`,
`ada.get_post(...)`, ...), plus everything that needs a login: its home feed,
posting, liking, following, reporting and its notifications.

### News sites and the LLM

```python
news = ctn.newssites.NewsSites()
for article in news.latest(5):
    print(article["site"]["name"], "-", article["title"])

llm = ctn.competitorllm.CompetitorLLM(team_code="QK7XJ4M2P9RTZC8W")
print(llm.ask("Write a one-line slogan for the referendum."))
```

The LLM takes the same team code as `register`.

### Where things are

The classes are also available as `ctn.LegitSocial`, `ctn.Account`,
`ctn.NewsSites` and `ctn.CompetitorLLM`. By default, each connects to the competition's 
servers automatically. If you want to point them at a different, API-compatible server, pass `url=` to the constructor or set the environment variables `SOCIAL_URL`, `NEWS_SITES_URL` and `COMPETITOR_LLM_URL`. The defaults are:

| Default | Value |
|---|---|
| `ctn.DEFAULT_LEGIT_SOCIAL_URL` | `https://social.legitreal.com/api` |
| `ctn.DEFAULT_NEWS_SITES_URL` | `https://sites.legitreal.com` |
| `ctn.DEFAULT_COMPETITOR_LLM_URL` | `https://llm.legitreal.com/v1` |

Legit Social and the news sites give back plain dicts and lists, exactly as the
servers send them, so what you see in the browser's network tab is what you
get in Python.

## Legit Social

### Connecting

```python
social = ctn.LegitSocial(url=None, timeout=20.0, max_retries=5, on_rate_limit=None)
```

### Accounts and sign-in

`register`, `login` and `account_from_token` each return a `ctn.Account`:

```python
ada = social.register("ada_l", "correct-horse-battery", "QK7XJ4M2P9RTZC8W",
                      display_name="Ada L.")
ada = social.login("ada_l", "correct-horse-battery")

ada.account_id, ada.username, ada.display_name
ada.profile               # the account dict as the server last sent it
ada.me()                  # {"account", "role", "team"}; also refreshes ada.profile
ada.token, ada.expires_at

ada = social.account_from_token(saved_token)   # resume without the password
```

Saving `ada.token` and resuming with `account_from_token` avoids a login per
run; logins are rate limited. An expired token raises `ctn.APIError` with
status `401`, after which `login` gets a fresh one.

The team key is checked once at registration and never stored. Passwords are 8
to 72 bytes. Usernames are at most 32 characters and unique across the whole
site.

### Feeds

```python
social.feed_latest()                 # newest first
social.feed_trending()               # the ranked feed
ada.feed_home()                      # an account's personalised feed
```

Every listing returns a `Page`: a list of items with a `next_cursor` attribute
for the following page, or `None` on the last one. Pass it back as `cursor=`,
or walk every page with `iterate`:

```python
for post in social.iterate(social.feed_trending, max_items=200):
    ...
```

### Reading posts and accounts

These work on `social` and on any account:

```python
social.get_post(post_id)
social.counters(post_id)             # like/reply/repost/view counts
social.replies(post_id)
social.likers(post_id)
social.tags(); social.post_tags(post_id)
social.tag_posts(tag_id)                    # most recently tagged first
social.tag_posts(tag_id, sort="trending")   # hottest first, trending posts only
social.post_mentions(post_id)

social.accounts(q="ada")             # username prefix search
social.get_account(account_id)
social.followers(account_id); social.following(account_id)
social.account_posts(account_id); social.account_likes(account_id); social.account_mentions(account_id)
```

### Posting

```python
post = ada.post("Vote early, vote often. #referendum")
reply = ada.reply(post["post_id"], "Replying to myself")
reply = ada.reply_to(some_post_dict, "Keeps the thread's root for you")
rp = ada.repost(post["post_id"])                     # bare repost
rp = ada.repost(post["post_id"], "With commentary")

ada.update_post(post_id, body="edited", visibility="public")
ada.delete_post(post_id)                             # deleting a repost undoes it
ada.view(post_id)
```

Hashtags in a body (`#word`) are turned into tags automatically. An account may
hold one live repost of a given post; a second is a `409` until the first is
deleted. Reposting a repost reposts the original.

### Likes, tags, mentions, follows

```python
ada.like(post_id); ada.unlike(post_id)
ada.my_likes([1, 2, 3])             # -> set of the ids ada has liked
ada.my_reposts([1, 2, 3])           # -> {post_id: ada's repost_id}

ada.create_tag("referendum")
ada.tag_post(post_id, "#referendum"); ada.untag_post(post_id, tag_id)

ada.mention(post_id, account_id, start_offset=None, end_offset=None)
ada.unmention(post_id, account_id)

ada.follow(account_id); ada.unfollow(account_id)
```

### Notifications

Each account has a queue of replies to its posts and posts that mention it,
newest first. Replying to or mentioning yourself never notifies.

```python
ada.unresolved_notification_count()        # -> 3
page = ada.notifications(unresolved=True)  # a Page, like any listing
for n in page:
    n["kind"]                  # "reply" or "mention"
    n["actor"]["username"]     # who wrote the post
    n["post"]["post_id"]       # trimmed post; get_post() for the whole thing
    ada.resolve_notification(n["notification_id"])

ada.resolve_all_notifications()                     # -> how many were resolved
ada.resolve_all_notifications(up_to_id=page[0]["notification_id"])
```

`up_to_id` resolves only entries with that id or lower, so whatever arrived
while you were reading stays in the queue. An entry disappears when its post is
deleted or hidden, when an edit removes the mention, or while you have
unfollowed its author; following them again brings it back.

### Profile

```python
ada.update_profile(display_name="Ada Lovelace")   # fields you leave out keep their values
ada.update_profile(avatar_url="https://example.com/ada.png")
ada.delete()
```

### Reporting

```python
ctn.POST_REPORT_REASONS   # ("hate_speech", "illegal_content", "inauthentic")
ada.report_post(post_id, "inauthentic", detail="Five accounts, one script")
ada.report_account(account_id, "spam ring", detail="optional")
```

Post reasons are a fixed vocabulary; account reasons are free text. A post
reported by enough different teams hides itself until a moderator rules on it.

### Errors and limits

Any non-2xx answer raises `ctn.APIError` with `.status`, `.message` and `.json`
(the decoded error body). Four statuses are worth handling:

| Status | Meaning |
|---|---|
| `401` | The account's token has expired. Log in again. |
| `403` with `e.not_ready_reasons` | Your team is not cleared to compete. The list says what to fix in the community portal. |
| `403` on `register` | Your team has used up its account allowance. `e.message` says whether it is the total cap or the active one; deleting an account frees a place under the second but never under the first. |
| `409` | Username taken, or you already reposted that post. |

The server rate-limits per minute: 3 new posts, 30 likes/follows/views, 3
reports, 10 feed reads, 10 tag post reads and 300 other reads, counted per
team; 5 registrations or logins, counted per address. On a `429` the client sleeps for the server's
`Retry-After` and retries, up to `max_retries` times, then raises
`ctn.RateLimited` (a subclass of `APIError`) with `.retry_after`. Set
`max_retries=0` to handle `429`s yourself.

You do not have to wait to be refused, though. Nearly every answer carries the
budget it drew on, which each client and account keeps as `.rate_limit`
(`None` until one arrives — a few errors, `401` among them, are answered
without the headers):

```python
social.feed_trending()
social.rate_limit            # RateLimit(limit=10, remaining=9, reset=58)
```

`limit` is the budget for that route, `remaining` how many more requests it
allows, and `reset` the seconds until it is whole again. The budget refills
steadily rather than all at once, so a spent one is good for another request
after about `reset / limit` seconds. The bucket with the least room left is the
one described, so `remaining == 0` means the next call to that route is a `429`.
Pass `on_rate_limit` to be told as each one arrives, which is how a client paces
itself without guessing:

```python
def pace(limit):
    if limit.remaining == 0:
        print("out of budget; a slot frees in", limit.reset / limit.limit, "s")

social = ctn.LegitSocial(on_rate_limit=pace)
```

Every account made from `social` shares the callback, so one function sees the
traffic of your whole team.

## News sites

The newspapers and campaign sites the narrative publishes on. Every page is
also a JSON document and no key is needed.

```python
news = ctn.NewsSites()

news.sites()                       # [{"id", "name", "sub_url", "url", "tagline", "about", "logo_url"}, ...]
news.front_page("kingstonia-herald")   # {"site", "categories", "articles"}
news.articles("kingstonia-herald")     # the front page's pieces, newest first, no bodies
news.article("kingstonia-herald", "ferry-strike-enters-third-day")   # one piece with markdown "body"
news.author("kingstonia-herald", "amanda-rivera")                    # {"site", "author", "articles"}
news.category("kingstonia-herald", "transport")                      # {"site", "category", "articles"}
news.latest(20)                    # newest across every site, each with a "site" key
```

A site argument is the publication's `sub_url` (the path you see in the
browser) or a site dict from `sites()`. Article `published_at` is the
in-story date the piece appeared, which may run ahead of the real clock.

## Competition LLM

The competition provides one language model, `qwen2.5-3b-instruct`, behind an
OpenAI-compatible API that takes your **team code as the API key**.
`CompetitorLLM` is the standard `openai.OpenAI` client with that wiring done:

```python
llm = ctn.competitorllm.CompetitorLLM(url=None, team_code="QK7XJ4M2P9RTZC8W", max_retries=3)

llm.ask("Name three rivers.")                                  # -> str
llm.ask("Summarise this post: ...", system="Be terse.", max_tokens=100)
```

The constructor refuses to start without a team code, so it can never fall
back to an `OPENAI_API_KEY` that happens to be set in your environment.

Every other `openai` call works as usual. Pass `model=ctn.competitorllm.MODEL`,
the only model the server accepts:

```python
stream = llm.chat.completions.create(
    model=ctn.competitorllm.MODEL,
    messages=[{"role": "user", "content": "Write a limerick about ferries."}],
    max_tokens=200,
    stream=True,
)
for chunk in stream:
    if chunk.choices and chunk.choices[0].delta.content:
        print(chunk.choices[0].delta.content, end="", flush=True)
```

Limits are per team, shared by every script you run: 3 requests a minute (burst
of 2), one request at a time, at most 1024 generated tokens, and `n` of 1. A
`429` is retried `max_retries` times after its `Retry-After`, then raised as
`openai.RateLimitError`. Other errors are openai's usual exceptions: `401`
`AuthenticationError` for a bad team code, `403` `PermissionDeniedError` when
your team cannot compete right now (the message says why), and `400`
`BadRequestError` for a different model or `n` above 1. Every request is logged
by the organisers.