# @three-ws/shipfeed

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/three-ws-shipfeed?style=flat&logo=github)](https://github.com/nirholas/three-ws-shipfeed/stargazers) [![License](https://img.shields.io/github/license/nirholas/three-ws-shipfeed?style=flat)](https://github.com/nirholas/three-ws-shipfeed/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/three-ws-shipfeed?style=flat)](https://github.com/nirholas/three-ws-shipfeed/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/three-ws-shipfeed/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/three-ws-shipfeed/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


Turn a repository's raw commit history into a release feed humans actually read, and join it to the changelog you already write.

Every project keeps two records of the same work. The commit log is complete and unreadable. The changelog is readable and incomplete. Nothing points one at the other, so nobody can answer the two questions that matter most about a release: *which code shipped this?* and *what shipped that nobody wrote up?*

`shipfeed` answers both. It parses commits, scores who each one is for, and matches release notes to the commits that produced them, without asking you to add a single line of metadata to your workflow. Every match carries a confidence score and the reasons behind it, so a wrong link is visible instead of implied.

Zero dependencies. Pure functions. Runs in Node, in a serverless handler, and in a browser.

```bash
npm install @three-ws/shipfeed
```

## The 30-second version

```bash
# The current repo, read from GitHub, printed as a report
npx @three-ws/shipfeed

# Any repo, linked against any changelog JSON
npx @three-ws/shipfeed --repo nirholas/three.ws --changelog https://three.ws/changelog.json

# Why did this commit get classified and linked the way it did?
npx @three-ws/shipfeed explain 65f3fe5

# Velocity, audience mix, and how much of the work was ever announced
npx @three-ws/shipfeed stats
```

```
ship log · nirholas/three.ws
200 commits · 45 releases · 135 linked · 68% coverage

2026-08-28  Your agent now tells you the important things in person
    d1f7b45 Feature · companion let the corner avatar walk on and deliver a user's own messages
    b019fc9 Feature · companion speak fresh unread notifications through the corner avatar
    20f50e8 Feature · notifications wire the avatar as a first-class delivery channel
```

## In code

```js
import { shipfeed } from '@three-ws/shipfeed';

const feed = await shipfeed({
	repo: 'nirholas/three.ws',
	changelogUrl: 'https://three.ws/changelog.json',
	siteUrl: 'https://three.ws',
	limit: 200,
	productScopes: ['forge', 'wallet', 'companion'],
});

for (const release of feed.releases) {
	console.log(release.title, '<-', release.stats.commits, 'commits');
	for (const commit of release.commits) {
		console.log('  ', commit.shortSha, commit.headline, `(${commit.confidence})`, commit.why[0]);
	}
}

console.log(`${Math.round(feed.stats.coverage * 100)}% of visible commits are accounted for`);
console.log(`${feed.ships.length} bursts of work were never written up`);
```

Nothing here needs a token for a public repository, though GitHub's unauthenticated limit is 60 requests an hour per IP. Pass `token` (or set `GITHUB_TOKEN`) to raise it.

## What it does, in four pieces

### 1. Parse

Full Conventional Commits 1.0.0, plus the three things real repositories have that the spec never covers: revert subjects, GitHub's merge subjects, and trailers that carry meaning.

```js
import { parseCommitMessage, headline } from '@three-ws/shipfeed';

const c = parseCommitMessage('feat(resilience)!: add a breaker\n\nBREAKING CHANGE: v1 callers must migrate');
// { type: 'feat', scope: 'resilience', breaking: true, description: 'add a breaker', ... }

headline(c); // 'Feature · resilience (breaking)'
```

A non-conventional subject is not thrown away. `Avatar Studio: 122 sliders` keeps its own convention (`Avatar Studio` / `122 sliders`), and a subject with no convention at all is reported honestly as `New commit` rather than mangled into one.

### 2. Classify

Each commit gets an audience and a signal score from 0 to 1, with a reason for every point awarded.

| audience | who it is for |
| --- | --- |
| `holder` | someone who uses or owns a piece of the product |
| `developer` | someone who builds against it |
| `internal` | someone who works on it |

```js
import { classify } from '@three-ws/shipfeed';

classify(commit, { productScopes: ['forge', 'wallet'] });
// {
//   audience: 'holder',
//   signal: 0.86,
//   noise: false,
//   reasons: [
//     { rule: 'type:feat', delta: 0.72, note: 'conventional type "feat"' },
//     { rule: 'scope:forge', delta: 0.12, note: 'named product scope' },
//     { rule: 'description:explanatory', delta: 0.06, note: 'description explains the change' },
//   ],
// }
```

Merge commits, `chore(deps):` bumps, and lockfile-only changes come back `noise: true`. That single flag is usually all a notification lane needs to stop being unreadable.

`productScopes` is the highest-leverage option in the package: naming the parts of your product a reader has actually seen is what lifts `fix(forge)` above `fix(build)`.

### 3. Link

Which commits produced which release note, without metadata.

An explicit `Changelog: <slug>` trailer always wins. Where there is none (which is almost everywhere), four signals decide:

1. **A time window.** An entry is written the day its work lands, so only commits inside `windowDays` are candidates.
2. **IDF-weighted term overlap.** Shared rare words ("meshopt", "settle", "fingerspelling") count; shared common ones ("agent", "the", "fix") do not. Similarity blends coverage of the entry with containment of the commit, so a one-line commit can still explain a paragraph-long note.
3. **Tag agreement.** An entry tagged `fix` prefers a `fix:` commit.
4. **Proximity.** Same-day work outranks work three days out.

```js
import { linkCommits } from '@three-ws/shipfeed';

const { byEntry, byCommit, orphans } = linkCommits(entries, commits, { threshold: 0.26 });

byCommit.get(sha);
// {
//   entryKey: '2026-08-27:Text-to-3D now has five independent image sources behind it',
//   score: 0.41,
//   reasons: ['shared terms: painter, concept, providers', 'type "fix" matches tag', 'before the entry by 0.4d'],
// }
```

A commit belongs to at most one entry; an entry may claim many. Everything unclaimed comes back in `orphans`, which is the interesting half: it is the work that shipped and was never written up.

### 4. Render

```js
import { renderMarkdown, renderRss, renderTerminal, renderReleaseTelegram } from '@three-ws/shipfeed';

renderMarkdown(feed);                                  // a RELEASES.md or a PR body
renderRss(feed, { siteUrl: 'https://three.ws/ship' }); // a feed reader
renderTerminal(feed, { color: true });                 // a terminal report
renderReleaseTelegram(feed.releases[0], { repo });     // one chat message, with a "shipped in 7 commits" link
```

## API

| export | what it does |
| --- | --- |
| `shipfeed(options)` | read a repo (and changelog) and return the unified feed |
| `buildShipFeed({commits, entries, ...})` | the same thing from data you already have |
| `parseCommitMessage(message)` / `parseCommit(githubCommit)` | conventional-commit parsing |
| `headline(commit)` / `summaryLine(commit)` | reader-facing title and description |
| `classify(commit, options)` / `filterByAudience(commits, min)` | audience and signal scoring |
| `linkCommits(entries, commits, options)` | changelog-to-commit provenance |
| `groupIntoShips(commits, options)` | cluster commits into bursts |
| `renderMarkdown` / `renderRss` / `renderTerminal` / `renderCommitTelegram` / `renderReleaseTelegram` | output |
| `fetchGitHubCommits` / `fetchChangelog` / `commitsFromGitLog` / `normalizeChangelog` | sources |
| `tokenize` / `buildIdf` / `entryKey` / `entrySlug` | the linking internals, exported for tuning |

Full types ship in `src/index.d.ts`.

### Feed shape

```jsonc
{
  "version": 1,
  "generatedAt": "2026-08-28T03:00:00.000Z",
  "repo": "nirholas/three.ws",
  "releases": [
    {
      "date": "2026-08-27",
      "title": "Text-to-3D now has five independent image sources behind it",
      "summary": "...",
      "tags": ["fix", "infra"],
      "url": "https://three.ws/changelog/2026-08-27-text-to-3d-...",
      "commits": [
        {
          "shortSha": "2771f41",
          "headline": "Feature · forge",
          "summary": "three more independent concept-image rungs",
          "audience": "holder",
          "signal": 0.86,
          "confidence": 0.41,
          "why": ["shared terms: painter, concept, providers"]
        }
      ],
      "stats": { "commits": 4, "authors": 1, "range": "2771f41..c6f0e52" }
    }
  ],
  "ships": [{ "id": "a1b2c3d..e4f5a6b", "title": "Fix · wallet", "commits": [] }],
  "stats": {
    "commits": 200, "linked": 135, "orphans": 65, "coverage": 0.68,
    "byAudience": { "holder": 124, "developer": 54, "internal": 22 },
    "velocity": [{ "date": "2026-08-27", "count": 61 }]
  }
}
```

## Working from a local clone

No network, no API limit:

```js
import { execFileSync } from 'node:child_process';
import { commitsFromGitLog, GIT_LOG_FORMAT, buildShipFeed } from '@three-ws/shipfeed';

const log = execFileSync('git', ['log', '-n200', `--format=${GIT_LOG_FORMAT}`], { encoding: 'utf8' });
const feed = buildShipFeed({ commits: commitsFromGitLog(log, { repo: 'o/r' }), entries });
```

The CLI does exactly this behind `--local`.

## CLI reference

```
shipfeed [feed]            build the unified release feed (default)
shipfeed explain <sha>     why one commit was classified and linked as it was
shipfeed stats             velocity, audience mix, and changelog coverage
shipfeed post              render what would go to a channel (add --send to deliver)

--repo <owner/name>        default: parsed from the git "origin" remote
--branch <name>            default: main
--limit <n>                commits to read (default 200)
--local                    read commits from the local clone with git log
--changelog <url|path>     changelog JSON to link commits against
--site <url>               site base URL used for permalinks
--scopes <a,b,c>           scopes that mean "product" when scoring signal
--min-audience <a>         holder | developer | internal (default internal)
--json | --markdown | --rss | --no-color
--telegram --chat <id> --token <t> --send     post mode; dry run without --send
```

`post` never sends anything without `--send`. Without it you get the exact messages that would have gone out, on stdout.

## Where this runs in production

three.ws publishes its own feed from this package:

- [three.ws/ship](https://three.ws/ship) renders it
- [three.ws/api/ship/feed](https://three.ws/api/ship/feed) serves it as JSON, Markdown, or RSS, and `?explain=<sha>` returns one commit's full reasoning
- the holder Telegram channel uses it twice: release announcements carry a "shipped in N commits" link, and the raw commit lane drops merge and lockfile noise before it posts

## License

Apache-2.0

<!-- three.ws:growth -->
## Support the project

If three-ws-shipfeed saves you time, **[star it on GitHub](https://github.com/nirholas/three-ws-shipfeed)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=three-ws-shipfeed%3A%20Turn%20a%20repo's%20raw%20commit%20history%20into%20a%20release%20feed%20humans%20actually%20read%3A%20conventional-commit&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-shipfeed) · [Share on Bluesky](https://bsky.app/intent/compose?text=three-ws-shipfeed%3A%20Turn%20a%20repo's%20raw%20commit%20history%20into%20a%20release%20feed%20humans%20actually%20read%3A%20conventional-commit%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-shipfeed) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-shipfeed) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-shipfeed&t=three-ws-shipfeed%3A%20Turn%20a%20repo's%20raw%20commit%20history%20into%20a%20release%20feed%20humans%20actually%20read%3A%20conventional-commit) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fthree-ws-shipfeed&title=three-ws-shipfeed%3A%20Turn%20a%20repo's%20raw%20commit%20history%20into%20a%20release%20feed%20humans%20actually%20read%3A%20conventional-commit)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/three-ws-shipfeed` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/three-ws-shipfeed/issues) or [start a discussion](https://github.com/nirholas/three-ws-shipfeed/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/three-ws-shipfeed)](https://github.com/nirholas/three-ws-shipfeed/graphs/contributors)

## Star history

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/three-ws-shipfeed&type=Date)](https://www.star-history.com/#nirholas/three-ws-shipfeed&Date)

<!-- /three.ws:growth -->

<!-- three.ws:ecosystem -->
## Part of three.ws

[three.ws](https://three.ws) is a platform for 3D AI agents with Solana wallets: avatars, a skill marketplace, x402 payments and more than seventy MCP servers. `@three-ws/shipfeed` is one package from it.

- Documentation site: https://nirholas.github.io/three-ws-shipfeed/
- npm: https://www.npmjs.com/package/@three-ws/shipfeed
- Canonical source: https://github.com/nirholas/three.ws (this repository is a generated mirror of `packages/shipfeed`)
- Agent-readable summary: [llms.txt](./llms.txt) and [AGENTS.md](./AGENTS.md)
- Issues and ideas: https://github.com/nirholas/three-ws-shipfeed/issues
<!-- /three.ws:ecosystem -->
