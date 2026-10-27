# Osysharp.ProblemReports

One app tells another about the problems it sees, and the other follows each one to the end. A **reporter** — a system
that watches something: its failed jobs, its alerts, its errors — names each problem by a key of its own and sends it
whole, as it now is, each time it moves. A **receiver** — a help desk, a bug tracker, an on-call board — keeps one
record per key, so a report sent twice changes nothing twice. Either side can be any Osy# app, and one app can be both.

Two kits, one per side: **`Osysharp.ProblemReports`** (this one) is the wire — the report, its steps, the receipt — and
the receiving side. **`Osysharp.ProblemReports.Sending`** is the reporting side. A receiver uses this kit alone; a reporter
uses both; one app that is both uses both.

Neither app's source names the other. The receiver's address, its key for the reporter, and the principal that reports
are each environment's settings and data.

## The receiver

```osy
use Osysharp.ProblemReports@0;

// Who may report: an integration principal (another system, by its API key) holding one role.
policy ReportsProblems => RoleGrant.Any(g => g.Member == user && g.Role == DeskRole.Reporter);

// What a report becomes: a ticket the first time its key is reported, the same ticket updated after.
public class DeskTickets : IProblemReceiver {
  public ProblemReportReceipt Receive(ProblemReport report) { … return new ProblemReportReceipt { Reference = "#" + t.Number }; }
}
app.ProblemReports = new ProblemReportsSetup { Receiver = new DeskTickets() };

// The route: PUT /api/rest/v1/problems/{key}
app.Apis = [ new RestApi("Problems") { Route = "problems", Auth = new ApiAuth { ApiKey = true },
  Endpoints = [ new Endpoint(ReceiveProblemReport) { Method = HttpMethod.Put, Path = "/{key}" } ] } ];
```

## The reporter

The reporting side is its own kit, **`Osysharp.ProblemReports.Sending`** (`kits/problem-reports-sending`), so a receiver
asks for nothing it does not use: this kit reaches no host and reads no secret. A reporter uses both:

```osy
use Osysharp.ProblemReports@0;
use Osysharp.ProblemReports.Sending@0 { egress "*"; secret "ProblemReceiverKey"; }   // or the receiver's host
```

and sets `app.ProblemReporting = new ProblemReportingSetup { Source = …, ReceiverUrl = Config.… };` — see its README.

## What it does

| | |
|---|---|
| **One request** | `PUT <receiver>/<key>`, body `{"Report": …}`, the receiver's key in `X-API-Key`, answered with a receipt: the receiver's reference for the record, whether this report made it, and its link. Both sides compile against the same classes; a reporter or receiver in another language sends or takes the same JSON |
| **The whole problem, each time** | evidence, how often, where, and its steps — triaged, fixed, released, verified gone, back again — so the last report a receiver took is the problem as it is |
| **Owed first, sent later** | marking a problem owed is one row in the work that moved it; nothing there calls the receiver. A pass asks for each owed report as it is then, so a problem that moved three times is sent once |
| **Never given up on** | a refusal is kept in the receiver's own words and tried again on a doubling wait. A receiver that does not answer ends the pass and holds back everything owed, so while it is down it costs one call per wait |
| **Off until configured** | with no receiver address or no key, nothing is sent and nothing fails; what is owed goes once both are set |
| **The receiver decides** | who may report (its policy, over principals its administrator made) and what a report becomes (its `IProblemReceiver`) |
| **A line for your page** | `ProblemReportLine(key)` says where the receiver keeps a problem, or what is owed and why |

Reference: `osy docs http-problem-reports-kit`.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.ProblemReports;

[Test(Observatory)]
void a_report_makes_one_record_and_a_repeat_updates_it() {
  var key = Api.KeyFor(Watcher);
  var first = Api.Put("/api/rest/v1/problems/dome-1", Body("dome-1", 1, Opened()), apiKey: key);
  Assert.Equal(200, first.StatusCode);
  var receipt = JsonSerializer.Deserialize<ProblemReportReceipt>(first.Body);
  Assert.True(receipt.Created);
  Assert.StartsWith("#", receipt.Reference);

  var again = Api.Put("/api/rest/v1/problems/dome-1", Body("dome-1", 2, Opened() + "," + Recurred()), apiKey: key);
  Assert.Equal(200, again.StatusCode);
  Assert.False(JsonSerializer.Deserialize<ProblemReportReceipt>(again.Body).Created);
  // The same report sent twice — a retry after a lost answer — adds no step twice.
  Api.Put("/api/rest/v1/problems/dome-1", Body("dome-1", 2, Opened() + "," + Recurred()), apiKey: key);

  runas (Vera) {
    Assert.Equal(1, LogbookEntry.Count());
    var e = LogbookEntry.Single(x => x.Key == "dome-1");
    Assert.Equal(2, e.Occurrences);
    Assert.Equal(3, e.Reports);
    Assert.Equal("https://watcher.test/dome", e.Link);
    Assert.Equal(2, LogbookStep.Count(s => s.Entry == e));
  }
}
```

From [`tests/receiving.test.osy`](tests/receiving.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.ProblemReports;

[Test(Observatory)]
void only_a_principal_the_receiver_admits_may_report() {
  Assert.Equal(401, Api.Put("/api/rest/v1/problems/dome-1", Body("dome-1", 1, Opened())).StatusCode);
  Assert.Equal(403, Api.Put("/api/rest/v1/problems/dome-1", Body("dome-1", 1, Opened()), apiKey: Api.KeyFor(Vera)).StatusCode);
  runas (Vera) { Assert.Equal(0, LogbookEntry.Count()); }
}
```

From [`tests/receiving.test.osy`](tests/receiving.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.ProblemReports;

[Test(Observatory)]
void a_report_without_its_title_or_under_another_key_is_refused() {
  runas (Watcher) {
    Assert.Throws<ValidationException>(() => ReceiveProblemReport("dome-1", new ProblemReport { Key = "dome-1", Title = " " }));
    Assert.Throws<ValidationException>(() => ReceiveProblemReport("dome-1", new ProblemReport { Key = "dome-2", Title = "The dome" }));
    Assert.Throws<ValidationException>(() => ReceiveProblemReport(" ", new ProblemReport { Title = "The dome" }));
  }
}
```

From [`tests/receiving.test.osy`](tests/receiving.test.osy).

## Learn more

- [The Osysharp.ProblemReports reference](https://osysharp.com/reference/http/problem-reports-kit/)
- [Osysharp.ProblemReports in the package catalogue](https://osysharp.com/packages/#osysharp-problemreports)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
