# Osysharp.ProblemReports.Sending

The reporting side of [`Osysharp.ProblemReports`](../problem-reports): an app tells a receiver — a help desk, a bug
tracker, an on-call board — about the problems it sees. The wire (the report, its steps, the receipt) and the receiving
side are in `Osysharp.ProblemReports`; this kit adds what only a reporter needs: the receiver's host, the secret
`ProblemReceiverKey`, and a delivery log.

## Use it

```osy
use Osysharp.ProblemReports@0;
use Osysharp.ProblemReports.Sending@0 { egress "*"; secret "ProblemReceiverKey"; }   // or the receiver's host

app.Config  = [ new Setting("ProblemDeskUrl") { Default = "" } ];   // set per environment; "" reports nowhere
app.Secrets = [ new Secret("ProblemReceiverKey") ];
policy SendsProblemReports => IsOperator;                          // who queues and sends

public class OvenProblems : IProblemSource {                       // what a problem's report says, as it now is
  public ProblemReport? ReportOf(string key) { … }
}
app.ProblemReporting = new ProblemReportingSetup { Source = new OvenProblems(), ReceiverUrl = Config.ProblemDeskUrl };

ProblemReportOwed(oven.Name);      // wherever a problem moves: one row, no call
SendOwedProblemReports();          // from your own schedule
```

## What it does

| | |
|---|---|
| **Owed first, sent later** | marking a problem owed is one row in the work that moved it; nothing there calls the receiver. A pass asks for each owed report as it is then, so a problem that moved three times is sent once |
| **Never given up on** | a refusal is kept in the receiver's own words and tried again on a doubling wait (`FirstWait`, at most `LongestWait`). A receiver that does not answer ends the pass and holds back everything owed, so while it is down it costs one call per wait |
| **Off until configured** | with no `ReceiverUrl` or no value for `ProblemReceiverKey`, nothing is sent and nothing fails; what is owed goes once both are set |
| **A line for your page** | `ProblemReportLine(key)` says where the receiver keeps a problem, or what is owed and why; `ProblemReportDelivery` is the log, read under `ReadsProblemReports` |

Reference: `osy docs http-problem-reports-kit`.

## Examples

Taken from this package's own tests, which compile and run on every release.

```osy
using Osysharp.Identity;
using Osysharp.ProblemReports;
using Osysharp.ProblemReports.Sending;
using Osysharp.Http;

[Test(NightShift)]
void with_no_receiver_key_nothing_is_sent_and_what_is_owed_is_kept() {
  runas (Otto) {
    JobFailed("focus-run", "the focuser did not answer");
    Assert.False(ProblemReceiverConfigured());
    Assert.Equal(0, SendOwedProblemReports());
    Assert.Equal(0, ReceivedPut.Count());
    var d = ProblemReportDelivery.Single(x => x.Key == "focus-run");
    Assert.Equal(ProblemReportState.Owed, d.State);
    Assert.Contains("no problem receiver is configured", ProblemReportLine("focus-run"));
  }
}
```

From [`tests/sending.test.osy`](tests/sending.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.ProblemReports;
using Osysharp.ProblemReports.Sending;
using Osysharp.Http;

[Test(NightShift)]
void an_owed_problem_is_sent_once_as_it_now_is_with_the_key() {
  Secret.ProblemReceiverKey.Stub("pk-logbook-test");
  runas (Otto) {
    JobFailed("focus-run", "the focuser did not answer");
    JobFailed("focus-run", "the focuser did not answer again");
    Assert.Equal(1, SendOwedProblemReports());
    Assert.Equal(1, ReceivedPut.Count());
    var put = ReceivedPut.Single();
    Assert.Equal("https://observatory.test/api/rest/v1/problems/focus-run", put.Url);
    Assert.Equal("pk-logbook-test", put.Key);
    var sent = JsonSerializer.Deserialize<ProblemReportBody>(put.Body).Report;
    Assert.Equal("focus-run", sent.Key);
    Assert.Equal(2, sent.Occurrences);
    Assert.Equal("the focuser did not answer again", sent.Message);
    Assert.Equal("Reported: #7.", ProblemReportLine("focus-run"));
    // Nothing moved since: a pass sends nothing.
    Assert.Equal(0, SendOwedProblemReports());
    Assert.Equal(1, ReceivedPut.Count());
  }
}
```

From [`tests/sending.test.osy`](tests/sending.test.osy).

```osy
using Osysharp.Identity;
using Osysharp.ProblemReports;
using Osysharp.ProblemReports.Sending;
using Osysharp.Http;

[Test(NightShift)]
void a_receiver_that_is_down_costs_one_call_and_gets_what_is_owed_once_it_answers() {
  Secret.ProblemReceiverKey.Stub("pk-logbook-test");
  runas (Otto) {
    ReceiverMood.Single().Status = 503;
    JobFailed("focus-run", "the focuser did not answer");
    JobFailed("dome-check", "the dome did not close");
    Assert.Equal(0, SendOwedProblemReports());
    // The first call that got no answer ended the pass.
    Assert.Equal(1, ReceivedPut.Count());
    var d = ProblemReportDelivery.OrderBy(x => x.OwedSince).First();
    Assert.Equal(ProblemReportState.Owed, d.State);
    Assert.Contains("HTTP 503", d.LastFailure);
    Assert.True(d.NextAttemptAt > DateTime.UtcNow);
    // Not due yet: nothing is tried.
    Assert.Equal(0, SendOwedProblemReports());
    Assert.Equal(1, ReceivedPut.Count());
    // It answers again, and the wait is over: both go.
    ReceiverMood.Single().Status = 200;
    foreach (var x in ProblemReportDelivery.ToList()) { x.NextAttemptAt = null; }
    Assert.Equal(2, SendOwedProblemReports());
    Assert.Equal(0, ProblemReportDelivery.Count(x => x.State == ProblemReportState.Owed));
  }
}
```

From [`tests/sending.test.osy`](tests/sending.test.osy).

## Learn more

- [The Osysharp.ProblemReports.Sending reference](https://osysharp.com/reference/http/problem-reports-kit/)
- [Osysharp.ProblemReports.Sending in the package catalogue](https://osysharp.com/packages/#osysharp-problemreports-sending)
- [The Osy# reference](https://osysharp.com/reference/) — the language, the runtime and every kit
- [`llms-full.txt`](https://osysharp.com/llms-full.txt) — the whole reference and specification as one Markdown file, for a coding agent

## Licence

MIT — see [`LICENSE`](LICENSE).
