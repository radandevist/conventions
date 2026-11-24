# Ports and adapters

Rules for every personal project (PublyApp, ShopKartOS, radan-ui, future ones), in TypeScript and in
.NET. Never DigitalPrevention. Written for agents; each rule says why. Owner decision of 2026-11-24,
taken while redesigning PublyApp's shared logger. The model is PublyApp's existing e-mail code:
`EmailService` → `IEmailSender` → `ResendEmailAdapter`.

## 1. Three layers: service → port → adapters

Every outside dependency (a library, an external API, a storage, the console) sits behind this shape:

```
your code ──► Service                       (what the code calls; does the shared work ONCE)
                 │
                 ▼
              IPort                         (an interface with ONE method)
                 │
                 ▼
              <Library><Function>Adapter    (one per library or API; only the outside call)
```

- **The service does the shared work once**: level filtering, redaction of personal data, consent,
  validation, retries. Why: written once, it cannot be forgotten by one adapter. If every adapter
  implemented the full contract, each would repeat that work, and one missing call would leak data.
- **The port has one method**: `ILogger.write(entry)`, `IEmailSender.SendAsync(request)`,
  `IAnalytics.send(event)`. Why: an adapter then costs a few lines and has nothing to get wrong.
- **An adapter only translates** the entry for its library or API. It holds no business rule, no `if`
  on the environment, no filtering and no redaction. Why: the adapters stay swappable and small.
- **Your code imports the service, never the port or an adapter.** Changing adapters touches one
  place, the wiring, and never a caller.
- **One adapter per library or external API.** Two classes that call the same API, for example one
  to connect an account and one to publish, are a defect. Merge them into one adapter used by both.
  Why: the API's URLs, errors and tokens must be known in one place.
- **Tests use a memory adapter**, such as `MemoryLoggerAdapter` or a fake sender. They assert on what
  the adapter received, never by spying on `console` or a global. Why: a spy still passes when the
  code stops writing, so it cannot prove "written, and without the secret".

## 2. Names

- **An adapter class is named library + function + `Adapter`**: `WinstonLoggerAdapter`,
  `JsonLoggerAdapter`, `MemoryLoggerAdapter`, `PostHogAnalyticsAdapter`, `ResendEmailAdapter`. Long
  is fine. Why: the name alone says which library, which job, and that it is an adapter. Do not use
  `…Provider`, `…Sink`, `…Transport`, `…Client` or `…Store` for an adapter.
- **A port is named `I` + function**: `ILogger`, `IEmailSender`, `IAnalytics`.
- **An environment variable names what it chooses**: `LOG_ADAPTER=winston|json`, not `LOG_SINK` or
  `LOG_FORMAT`. Why: an adapter has exactly one output format, so the variable picks the adapter.

## 3. Files

- **TypeScript files are in kebab-case**, with the adapters in an `adapters/` folder next to the
  service:

  ```
  lib/logger/
    logger.types.ts                    ILogger, LogEntry, LogLevel, LogMeta
    logger.ts                          class Logger (the service)
    log-redaction.ts
    log-level.ts
    index.ts                           export const logger + adapter choice
    adapters/
      winston-logger-adapter.ts        class WinstonLoggerAdapter implements ILogger
      json-logger-adapter.ts           class JsonLoggerAdapter implements ILogger
      memory-logger-adapter.ts         class MemoryLoggerAdapter implements ILogger
  ```

- **C# has one file per type, named exactly like the type** in PascalCase: `ResendEmailAdapter.cs`,
  `IEmailSender.cs`. Why: this is the .NET norm, and each language keeps its own convention.

## 4. Example

TypeScript:

```ts
export interface ILogger {
  write(entry: LogEntry): void;
}

export class Logger {
  constructor(private readonly adapter: ILogger, private readonly minLevel: LogLevel) {}

  debug = this.at('debug');
  info = this.at('info');
  warn = this.at('warn');
  error = this.at('error');

  private at(level: LogLevel) {
    return (message: string, meta?: LogMeta) => {
      if (rank(level) < rank(this.minLevel)) return;
      this.adapter.write({ level, time: new Date(), message: redactText(message), meta: meta && redactMeta(meta) });
    };
  }
}

export class WinstonLoggerAdapter implements ILogger {
  constructor(private readonly winston = createWinston()) {}
  write(entry: LogEntry) {
    this.winston.log(entry.level, entry.message, entry.meta);
  }
}
```

C#:

```csharp
public interface IEmailSender {
	Task<EmailSendReceipt> SendAsync(EmailRequest request, CancellationToken ct);
}

public class ResendEmailAdapter(IResendEmailClient client) : IEmailSender {
	public Task<EmailSendReceipt> SendAsync(EmailRequest request, CancellationToken ct) =>
		client.SendAsync(request.ToResend(), ct);
}
```
