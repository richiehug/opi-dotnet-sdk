# Integrating the OPI .NET SDK

This guide covers installation, configuration, terminal operations, result handling, recovery, and diagnostics. For platform requirements, see the [README](README.md). The [generated C# API reference](https://richiehug.github.io/opi-dotnet-sdk/) lists every public type, member and overload .

## Add the SDK to an application

### Local DLL

Choose **Full** for Windows .NET 10 (`net10.0`) or **Lite** for backward compatibility with .NET Framework 4.8/4.8.1 (`net48`). Both expose the same `Opi.Sdk` payment API. Applications consume events and results and control their own transaction experience.

Copy `Opi.Sdk.dll` and its XML IntelliSense file from the chosen ZIP to `libs/`. For Full, add this reference:

```xml
<ItemGroup>
  <Reference Include="Opi.Sdk">
    <HintPath>libs/Opi.Sdk.dll</HintPath>
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

For **Lite**, copy **all** DLLs from its ZIP, including the Microsoft dependencies. Reference the complete set instead of the single Full reference:

```xml
<PropertyGroup>
  <TargetFramework>net48</TargetFramework>
  <AutoGenerateBindingRedirects>true</AutoGenerateBindingRedirects>
  <GenerateBindingRedirectsOutputType>true</GenerateBindingRedirectsOutputType>
</PropertyGroup>
<ItemGroup>
  <Reference Include="libs/*.dll">
    <Private>true</Private>
  </Reference>
</ItemGroup>
```

For older project formats, use **Add Reference → Browse** to select those DLLs and enable automatic binding redirects in the executable project. Deploy the generated application `.config` file as well. Test with the cashier's existing dependency versions; do not overwrite its Microsoft DLLs blindly.

Full uses the .NET 10 runtime; Lite uses the installed .NET Framework 4.8/4.8.1 runtime and its bundled dependencies. Distribution remains DLL-only; integrators do not need a NuGet package. Do not reference Full and Lite together.

## Initialize the SDK

```csharp
using Opi.Sdk;

var options = new OpiOptions
{
    TerminalHost = "192.168.0.69",
    WorkstationId = "CASHIER-01",
    PayChannelPort = 4100,
    DeviceChannelPort = 4102,
    Language = "de",
    DataDirectory = Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "YourCashier", "Opi"),
    Receipts = new ReceiptHandling
    {
        MerchantReceipt = ReceiptHandlingMode.Available,
        CustomerReceipt = ReceiptHandlingMode.Available
    }
};

await using var terminal = new OpiClient(options);
```

The examples use modern C# syntax. Older C# 7.3 applications can use ordinary object initializers for `OpiOptions` and `ReceiptHandling`; `required` and `init` are not required to configure the SDK. Replace `await using` with explicit asynchronous cleanup:

```csharp
var terminal = new OpiClient(options);
try
{
    var result = await terminal.PaymentAsync(1250, "CHF");
    // Handle and persist result, including both receipt copies.
}
finally
{
    await terminal.DisposeAsync();
}
```

In a real cashier retain the client for the terminal's lifetime and dispose it only after outstanding operations finish. `CloseAsync()` is the terminal close-day operation, not client disposal. A .NET Framework 4.8/C# 7.3 consumer is compiled as part of SDK checks.

Retain one instance per terminal. Use a stable workstation ID and storage directory across application restarts. The terminal's callback configuration must point to the cashier machine. Configure Windows Firewall to allow callbacks from the terminal on the device port. A callback listener is created internally during each exchange, and callback peers are restricted to the resolved terminal address. This is raw OPI TCP on a trusted shop network; it is not an internet endpoint.

Socket closure fails promptly. Loss of the connected local network address ends the exchange after `ConnectivityLossGracePeriod` (5 seconds by default), even if another adapter remains connected. Silent remote outages can still last until TCP detects failure or `OperationTimeout` expires. A valid final approval received before the exchange ends remains `Success`, even after a temporary outage. If the request may have been sent and no reliable final response arrives, retain `Unknown` and reconcile before another payment.

Abort has an independent `AbortTimeout` of 10 seconds, capped by `OperationTimeout`. When the original operation completes, its remaining abort wait is cancelled so it cannot delay the financial result. Lost abort responses remain `ABORT_RESULT_UNKNOWN` when transmission may have occurred. An abort acknowledgement does not prove cancellation; close progress from the original operation's completion.

Do not create competing instances with different host aliases or data directories for the same terminal. Multiple terminals on a single cashier need explicitly coordinated, distinct callback ports and matching terminal configuration. This initial SDK is intended for one owner per terminal.


## Payments

```csharp
var payment = await terminal.PaymentAsync(1250, "CHF");
```

Amounts are positive integer minor units. CHF 12.50 is `1250`. Currency codes are normalized and validated; conversion also supports zero-, three- and four-decimal currencies. Merchant and authorization references are separate. Optional merchant references accept 1–40 ASCII letters, digits, underscores or hyphens, with no spaces, and are sent as `POSdata/MerchantReference`. Authorization references retain their 1–20 character limit. Reverse-last has no amount or reference and can only reverse the terminal's last eligible transaction.

An ordinary operational failure is returned as `OpiResult`. Invalid SDK configuration and use after disposal can throw. Persist each financial result against its sale using `CorrelationId`. A final `Amount` includes any positive `Tip` reported by the terminal.

## Refunds

```csharp
// Unreferenced refund; no original authorization identifier is needed.
var refund = await terminal.RefundAsync(1250, "CHF");
// Referenced refund, when supported and the original authorization is available.
var referencedRefund = await terminal.RefundAsync(1250, "CHF", authReference: "605224");
// Unreferenced refund with a merchant reference; the same rules apply as for payments.
var refundWithReference = await terminal.RefundAsync(new PaymentRequest(1250, "CHF", "RETURN-1001"));
```

Reversals carry no merchant reference. The terminal does not echo the merchant reference; `Transaction.Reference` returns the value that was sent. Reprints cannot return it.

Use a positive amount in minor units. The demo application always uses unreferenced refunds; the SDK supports both forms.

## Reversals

```csharp
var reversal = await terminal.ReversalAsync();
```

This reverses the terminal's last eligible transaction. It does not accept an amount or an arbitrary transaction identifier.

## Handle results

```csharp
if (payment.IsSuccessful)
{
    // Complete the sale and store payment.Transaction and its receipts.
}
else if (payment.Status == ResultState.Unknown)
{
    // Keep the sale unresolved. Do not retry the payment automatically.
}
else
{
    // Display the outcome using Status, Error and ErrorCode.
}
```

### Payment method

`Transaction.PaymentMethod` is the terminal's brand or payment-method label. It is optional; do not infer approval from its presence. Use `Status` to decide the outcome.

### Tip results

`Transaction.Amount` includes a positive `Transaction.Tip` reported by the terminal. Both use minor units. Do not add the tip a second time.

### DCC results

`Transaction.Dcc` optionally contains `Offered`, `Accepted`, `Amount`, `Currency`, `CurrencyCode`, `ExchangeRate` and `MarkupPercentage`. Preserve the terminal's strings for display; do not recalculate settlement totals from them.

## Receipt handling

```csharp
string? merchant = payment.Transaction?.Receipts?.MerchantReceipt?.Content;
string? customer = payment.Transaction?.Receipts?.CustomerReceipt?.Content;
```

**Recommended:** configure both copies as `ReceiptHandlingMode.Available`, as in the initialization example. This supports printerless terminals and lets the ECR store receipts before printing. `PrintLocal` requires a working terminal printer; otherwise the terminal can return `DeviceUnavailable` and leave a pending receipt. Both copies default to `Available`; select `PrintLocal` explicitly only when the terminal should print them.

The same receipt handling applies to terminal operations that can produce a merchant report, including CloseDay, TransmitTrx, ContactAcq, ContactTMS, ActivateTerminal, DeactivateTerminal, Login and Logoff. With `Available`, a returned report is available at `result.Transaction?.Receipts?.MerchantReceipt`; `PrintLocal` asks the terminal to print it.

This includes GetInfo: terminals such as the RX5000 answer `InfoAsync()` with an empty response and return the terminal details as an info receipt. The receipt is returned like other service receipts, and `result.Terminal` is filled from it where the response itself has no value: `TerminalId`, `Hardware` (model), `FirmwareVersion`, `SerialNumber`, `PartNumber`, `TerminalIp`, `MacAddress`, `AppVersion` and `Ep2Version`. These fields are read from English receipt labels; with another terminal language the receipt text is still returned. GetInfo can take several minutes while the terminal checks its server connections. `result.Terminal` also includes `TerminalId` and `Stan` from the response's `Terminal` element.

`PrintLocal` means the payment terminal prints. `Available` means printable text is returned to the cashier. Each copy is configured independently. Missing copies are normal if the terminal did not generate them or the mode did not request their return.

Receipt content is plain text, with line breaks and leading alignment spaces preserved. Render it in a monospace font. The cashier owns printing and durable storage. A receipt-only reprint has `Transaction.Receipts` but may have no amount or payment details.

## Events and cancellation

```csharp
terminal.OperationEvent += (_, e) =>
{
    if (e.Kind == OperationEventKind.TerminalMessage)
    {
        // Queue e.Message for application handling.
    }
        // Update the ORIGINAL sale using e.Result.CorrelationId.
    }
    if (e.Kind == OperationEventKind.ReceiptCaptured)
    {
        // Especially useful for receipts recovered by a reprint operation.
        // Keep e.Operation and e.CorrelationId; do not attach an old ticket to a new sale.
    }
};

var abort = await terminal.AbortAsync();
```

Events are delivered in order on a background worker. Slow or throwing application event handlers do not delay device acknowledgements. Events may arrive just after the awaited result; the result already contains its final receipts. Do not synchronously wait for SDK disposal from an event handler.

Abort bypasses the regular operation lock and uses a separate control connection. Attempts are delayed and limited. An abort response does not override an approved payment. If the abort response is lost, the abort call returns uncertainty and the financial operation continues resolving its own result. Do not cancel a local task and assume the payment was cancelled.

## Transaction state and reconciliation

**SDK owns OPI communication; ECR owns transaction state and reconciliation.**

The SDK keeps no persistent transaction ledger or cross-restart transaction lock. An overlapping operation is rejected with `PENDING_TRANSACTION` while an exchange is active. Save the transaction intent, terminal identity, request ID, result and receipts in your ECR.

`PaymentAsync()` → `PRINTLASTTICKET` → ECR decides → `ReprintAsync()` if required → ECR explicitly starts a new payment.

`PRINTLASTTICKET` is returned unchanged. The SDK does not reprint, reconcile an older transaction, check readiness, or retry the new payment automatically. A successful reprint does not start another payment.

If a financial request may have been sent but its final response is lost or unusable, the SDK returns `UNKNOWN` / `RESULT_UNKNOWN`. This is neither approval nor proof of failure. Keep the transaction unresolved and **do not blindly retry**: reconcile using the terminal/acquirer records and any explicitly requested evidence. A connection failure before transmission returns a communication/connection error. A definitive terminal response preserves its outcome and error code.

`RepeatLastMessageAsync()` explicitly requests the terminal's last registered message. `ReprintAsync()` explicitly requests its last receipt and available transaction details using the configured receipt handling. These are distinct commands; neither calls the other. The ECR must correlate their evidence with the original transaction, especially if another ECR has used the terminal. Neither command updates an SDK ledger.

Every result carries `RequestId`, the OPI RequestID assigned to the call, including failed, unknown and locally rejected results. Store it with the transaction from the Started event onward. A local validation or busy rejection has an assigned ID but sends no request and emits no operation events.

The status of a `RepeatLastMessageAsync()` result describes the RepeatLastMessage command itself. Whenever the terminal returns one, `OriginalHeader` identifies the last payment, refund or reversal (`RequestId`, `RequestType`, `WorkstationId`), with its raw `OverallResult` and the mapped `Status` and `ErrorCode`, whatever that outcome was. Match `OriginalHeader.RequestId` to the original transaction's `RequestId` before using its outcome. Without an `OriginalHeader`, keep the original transaction unresolved.

A reprint returns its own `RequestId`, not the original transaction's; match reprinted receipts using `Transaction.AuthReference`, `ApprovalCode`, amount and date. When receipts are requested as `Available` and the terminal reports success for TicketReprint without sending any receipt or journal output, the result is `TerminalError` with `NO_RECEIPT_RETURNED`; an empty GetInfo is reported as `NO_INFO_RETURNED`. Neither resolves an unknown transaction.

`RestartTerminal` (`ResetAsync()`) is acknowledged before the terminal restarts. Wait until the terminal is available again, for example with `StatusAsync()`, before sending the next request.

An `AbortRequest` acknowledgement does not prove that a financial transaction was cancelled. Keep awaiting the original financial result. If that response is lost, its outcome remains unknown; do not infer cancellation from an abort acknowledgement or closed socket.

## File logging and deployment

`LoggingEnabled = true` enables bounded daily files in `DataDirectory/logs`. Logs include configured ports/timeouts, connection stages, socket error codes, request/response byte counts, elapsed times, lifecycle events, receipt type/length, outcome/error codes. Normal excludes XML and receipt/message content. Extended includes sanitized OPI XML and error details. `Logger` optionally receives those same diagnostics. Diagnostic writes run on the background event queue rather than delaying callback acknowledgements; dispose the SDK to flush queued entries. A logging failure does not change a payment result.

Deploy the selected `Opi.Sdk.dll` alongside the cashier. Lite also needs its dependency DLLs and the host's binding-redirect configuration; .NET Framework 4.8/4.8.1 must be installed. A Full .NET 10 cashier can be published self-contained. No separate OPI Proxy process is needed.

## API overview

Namespace: `Opi.Sdk`.

## Terminal operations

All terminal calls return `Task<OpiResult>` unless indicated otherwise.

| Method | Purpose |
| --- | --- |
| `PaymentAsync(amount, currency, reference?)` / `PaymentAsync(PaymentRequest)` | Payment |
| `RefundAsync(amount, currency, authReference?)` | Referenced or unreferenced refund |
| `RefundAsync(PaymentRequest)` | Unreferenced refund with an optional merchant reference |
| `ReversalAsync()` | Reverse last eligible transaction |
| `AbortAsync()` | Out-of-band abort of active payment/refund |
| `ActivateAsync()` / `DeactivateAsync()` | Activate/deactivate terminal |
| `LoginAsync()` / `LogoffAsync()` | OPI login/logoff variants |
| `InfoAsync()` | GetInfo |
| `StatusAsync()` | GetStatus |
| `ReprintAsync()` | TicketReprint |
| `SubmitAsync()` | TransmitTrx |
| `CloseAsync()` | CloseDay |
| `ConfigAsync()` | ContactTMS |
| `InitAsync()` | ContactAcq |
| `ResetAsync()` | RestartTerminal |
| `RepeatLastMessageAsync()` | Read the terminal's last message; `OriginalHeader` identifies the last payment, refund or reversal |
| `DisposeAsync()` | `ValueTask`; release SDK ownership after operations |

## Result and event models

`OpiResult` contains `Status`, `Error`, `ErrorCode`, `CorrelationId`, `RequestId`, optional `Transaction`, optional `Terminal`, optional `OriginalHeader` (RepeatLastMessage only) and `IsSuccessful`.

`TransactionResult` contains `Reference`, `Amount`, `Currency`, `Tip`, `PaymentMethod`, `MaskedCardNumber`, `AuthReference`, `ApprovalCode`, `TransactionDate`, `AcquirerId`, `Receipts`, and `Dcc`. Fields are optional and depend on terminal evidence. `ReceiptDetails` separates `MerchantReceipt` and `CustomerReceipt`; `Receipt.Content` is plain text.

`OperationEvent` contains `Operation`, `CorrelationId`, `Kind`, `RequestId`, and optional `Message`, `ReceiptType`, `Receipt`, and `Result`. Kinds include Started, Connected, TerminalMessage, ReceiptCaptured, DccCaptured and Completed.

`OpiOptions` contains the client configuration. The client snapshots these settings when constructed; changing the options afterwards does not reconfigure an existing client. Supply `TerminalHost` and `WorkstationId`. Optional settings cover ports, language, receipts, force acceptance, PAN request preference, timeouts, diagnostic directory, logging. See configuration defaults below and the generated API reference for the full property list.


## Result states

| State | Meaning |
| --- | --- |
| `Success` | Operation completed successfully |
| `Declined` | Terminal/acquirer decline evidence |
| `Aborted` | Completed cancellation |
| `CommunicationError` | Connection or exchange problem; examine the error and operation context |
| `TerminalError` | Terminal or SDK operational error |
| `InvalidRequest` | Invalid input or a request blocked before execution |
| `InProgress` | Terminal is busy or still processing |
| `Unknown` | No reliable financial outcome; reconcile before another attempt |

`OpiError` provides stable categories: Failure, Aborted, ReprintRequired, DeviceUnavailable, TerminalBusy, LoginRequired, ResultUnknown, RecoveryMismatch, ConnectionError, OperationTimeout, InvalidRequest, InvalidOpiResponse, StorageError and Other. `ErrorCode` preserves a more specific terminal or SDK reason.

## Configuration defaults

| Option | Default |
| --- | --- |
| `TerminalHost`, `WorkstationId` | Required |
| `PayChannelPort`, `DeviceChannelPort` | `4100`, `4102` |
| `Language` | `en`; also supports `de`, `fr`, `it` |
| Merchant/customer receipt mode | `Available` for both |
| `ForceAcceptance` | `false` |
| `RequestFullPan` | `null` (no explicit preference) |
| `ConnectTimeout` | 10 seconds |
| `OperationTimeout` | 5 minutes per exchange |
| `ConnectivityLossGracePeriod` | 5 seconds |
| `AbortRetryDelay`, `MaxAbortAttempts` | 1.5 seconds, 3 |
| `DataDirectory` | Local application data / Opi / Sdk |
| `LoggingEnabled`, `LogLevel`, `MaxLogStorageBytes` | `true`, `Normal`, 1 MiB |
| `Logger` | `null` |

TCP keepalive starts probing after 5 seconds of socket inactivity. The operation deadline closes the exchange socket even if the terminal still responds to TCP probes. Reconcile an unknown payment by matching its request ID before attempting the sale again.

The API website is generated from the SDK projects and XML comments during each SDK build. Publishing a release copies the matching version into the public website and updates its landing page. New public signatures therefore appear automatically; maintainers update XML comments when behavior changes.

## Repeat Last Message

```csharp
var last = await terminal.RepeatLastMessageAsync();
```

This explicitly reads the terminal's last message for ECR inspection. Its correlation ID and `RequestId` belong to this lookup, not an earlier transaction, and its `Status` describes the lookup itself. `last.OriginalHeader` carries the original request's `RequestId`, `RequestType`, `WorkstationId`, raw `OverallResult` and mapped `Status`/`ErrorCode`. Match `OriginalHeader.RequestId` to a stored transaction before using its outcome; without an `OriginalHeader`, keep that transaction unresolved.

## Storage locations

`OpiOptions.DataDirectory` selects the SDK diagnostic directory; daily logs are in `DataDirectory/logs`. The default is the current user's local application data directory followed by `Opi/Sdk`. The ECR owns transaction and receipt persistence. No SDK transaction ledger is written or loaded.

### Diagnostic levels and application errors

`LoggingEnabled` defaults to `true`. Set it to `false` to opt out. When enabled, `LogLevel = OpiLogLevel.Normal` (default) records SDK method names, labeled request parameters and configuration, final response status/amount/error codes and operation summaries; `OpiLogLevel.Extended` also records sanitized OPI communication. Extended diagnostics are limited to SDK communication and SDK errors. Application exceptions and crashes are outside SDK logging.


When the ECR requests `ReversalAsync()`, the SDK automatically confirms the terminal’s `GetConfirmation` prompt for that reversal. Receipt confirmations retain their existing automatic handling; other card-consent prompts are not automatically confirmed.

`InfoAsync()` sends OPI `GetInfo`. A terminal can emit diagnostic printer reports and still return `Failure`; the SDK preserves that terminal result. Use `StatusAsync()` (`GetStatus`) for current terminal identity and readiness. An idle status is not proof that a pending receipt was cleared.

Log storage defaults to a 1 MiB total limit. Oldest daily files and then oldest complete entries are removed first; the newest entries are retained even when a single day reaches the limit.
