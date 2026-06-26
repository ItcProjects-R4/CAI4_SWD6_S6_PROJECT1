[README.md](https://github.com/user-attachments/files/29380650/README.md)
# Booking.com Automation 

This is the merged result of everyone's individual automation submissions
for testing **https://www.booking.com**. It is a single Maven **multi-module**
project so the whole team's work builds and runs together from one root.

## Why multi-module instead of one flat package?

Several teammates independently wrote classes with the **same names**
(`HomePage`, `BasePage`, `SignInPage`, `ConfigReader`, `DriverManager`...)
but **different method signatures and constructors** — because each person
built their own mini-framework rather than agreeing on one shared base first.

Forcing all of that into a single `com.booking.pages.HomePage` class would
silently have broken methods that other files depend on (e.g. one
`HomePage` takes a `WebDriver` in its constructor, another uses a no-arg
constructor backed by a singleton `DriverManager`). Renaming methods to
"merge" them would change behavior no one asked for.

So instead, each teammate's submission became its own **Maven module** with
its own package namespace. Nothing was deleted; everything still runs, and
modules can be run independently or all together.

## Module map

| Module                | Package                  | Covers                                                                 | Source submissions merged |
|------------------------|---------------------------|-------------------------------------------------------------------------|----------------------------|
| `hotels-core`          | `com.booking.hotels`      | Home page, Sign-in (OTP via Mailosaur), Search Results + filters, Property Details, My Account, Confirmation | "Tester1" + "myaccount" (myaccount is a superset of Tester1 — same classes, plus sign-in/account additions) |
| `room-details`         | `com.booking.roomdetails` | Hotel page room list, "Room Details" modal (images, facilities, price, reserve button) | "booking-room-details-automation" |
| `flights`              | `com.booking.flights`     | Flight search, full flight booking E2E, My Bookings, Mailosaur email verification | "booking-automation" (the flights/E2E one — two upload copies of this were identical, only one kept) |
| `car-flight-filters`   | `com.booking.cff`         | Flight result filters, Car Rental filters, Checkout/Payment tests (valid card, cancel, invalid card, pay-at-hotel) | "booking-automation" (the filters/checkout one) |

## Running tests

```bash
# Everything, all modules
mvn test

# Just one module
mvn -pl hotels-core test
mvn -pl room-details test
mvn -pl flights test
mvn -pl car-flight-filters test
```

Each module also has its own `testng.xml` under `src/test/resources/` if
you want to run via TestNG directly instead of Maven.

## What changed during the merge (and why)

1. **Duplicate upload removed.** Two of the uploaded zips
   (`booking-automation.zip` and `booking-automation__2_.zip`) were
   byte-for-byte identical. Only one copy was kept (now the `flights`
   module).

2. **Anti-bot "stealth" code removed from `hotels-core`'s `BaseTest`.**
   The "myaccount" submission's `BaseTest` used Chrome DevTools Protocol
   commands to hide `navigator.webdriver`, spoof the WebGL/canvas
   fingerprint, and override the user-agent — i.e. active evasion of
   booking.com's own bot-detection. That isn't needed for functional UI
   testing, so it was intentionally left out of the merge. The rest of
   that file's improvements (longer page-load timeout, explicit wait for
   the search bar instead of a fixed sleep, cookie-banner dismissal) were
   kept since they're useful and unrelated to evasion.

3. **Hardcoded Mailosaur API keys removed.** Two different submissions
   each hardcoded a *different* Mailosaur API key + server ID directly in
   source (`Login.java` and `MailosaurOtpReader.java`). Those are
   live credentials and should never be committed to source control.
   The merged `MailosaurOtpReader` (in `hotels-core`) now reads
   `MAILOSAUR_API_KEY`, `MAILOSAUR_SERVER_ID`, and `MAILOSAUR_DOMAIN`
   from environment variables only. Set them before running sign-in
   tests:
   ```bash
   export MAILOSAUR_API_KEY=your_key_here
   export MAILOSAUR_SERVER_ID=your_server_id
   export MAILOSAUR_DOMAIN=your_server_id.mailosaur.net
   ```
   The `flights` and `car-flight-filters` modules already used
   placeholder values (`YOUR_MAILOSAUR_API_KEY`, etc.) in
   `config.properties` / constants — those were left as placeholders for
   you to fill in locally; nothing real was ever in them.

4. **`PropertyDetailsPage` merged from two near-duplicate versions.**
   One version checked the page loaded via the URL containing `/hotel/`;
   a separately uploaded version checked via the visible hotel-name
   heading. The merged version checks both, since each catches a
   different kind of failure (URL check still "passes" on a blank/broken
   page; element check confirms real content rendered).

5. **`BasePage`/retry logic.** Where one submission had a simple
   `click()`/`type()` and another added stale-element retry + a
   JavaScript-click fallback for the same class, the sturdier version was
   kept (no functionality was lost — it's a strict improvement).

6. **Not carried over:** a `TimingTest.java` from the "myaccount" upload
   was a manual debugging script (prints timings, no real assertions) and
   wasn't included as a test case — happy to add it back if it's wanted
   for local debugging.

## Known gaps / things to double check before a real run

- All CSS/XPath selectors in every module are **as the original authors
  wrote them**. booking.com changes its DOM/A-B tests frequently — several
  of the original files even say so in their own comments
  (`room-details`'s `HotelPage`/`RoomDetailsPage` especially). Re-verify
  selectors with DevTools before trusting a CI run.
- `flights` and `car-flight-filters` use placeholder test account
  credentials (`your_test_email@example.com` / `YOUR_MAILOSAUR_API_KEY`)
  — fill in real sandbox/test values before running those suites.
- This merge was put together and sanity-checked (package structure,
  cross-class method calls, syntax) in a sandboxed environment without
  internet access to Maven Central, so `mvn test` itself could not be run
  here. Run `mvn -pl <module> test-compile` yourself first to confirm a
  clean build in your environment before a full test run.
