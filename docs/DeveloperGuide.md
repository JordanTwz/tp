# Developer Guide

## Acknowledgements

{list here sources of all reused/adapted ideas, code, documentation, and third-party libraries -- include links to the original source as well}

## Design

This section describes the overall architecture and explores the core classes of FinTrack.

### FinTrack Module (`FinTrack.java`)

`FinTrack` (`src/main/java/seedu/fintrack/FinTrack.java`) serves as the main entry point and the central controller of the application. It is responsible for managing the application's lifecycle and coordinating the interactions between the user interface (`Ui`), the business logic (`FinanceManager`), and the input processor (`Parser`). The `main` method executes a simple, continuous 'Read-Evaluate-Print' loop (REPL).

How the `FinTrack` component works:

1. The application starts by welcoming the user (via `Ui.printWelcome()`) and initialising the `FinanceManager`, which holds the application's state (all incomes, expenses and budgets).
2. It waits for user input using `Ui.waitForInput()`.
3. The input is parsed as follows:
   - The command word (e.g. `add-expense`) is extracted using `Parser.returnFirstWord()`.
   - A `switch` statement is used to route the command word to the appropriate logic block.
   - For commands that take no arguments (like `list`, `help` and `bye`), it uses the `hasUnexpectedArguments()` helper to validate the input before executing.
   - For commands that take arguments (like `add-expense`, `delete-income`, `budget`), it delegates the parsing of the entire input string to the `Parser` class (e.g. `Parser.parseAddExpense(input)`).
4. The relevant command is executed as follows:
   - If the `Parser` successfully returns a valid object (like an `Expense` or an `int`), `FinTrack` passes this object to the `FinanceManager` to perform the business logic (e.g. `fm.addExpense(expense)`).
   - Based on the result from `FinanceManager`, it then calls the appropriate `Ui` method to show success (e.g. `Ui.printExpenseAdded(expense)`).
   - Any `IllegalArgumentException` or `IndexOutOfBoundsException` (from `Parser` or `FinanceManager`) is caught within the loop. The error message is retrieved (`e.getMessage()`)
     and printed to the user via `Ui.printError()`.
5. The loop continues until the user enters the `bye` command (`Ui.EXIT_COMMAND`).

The above flow is illustrated by the sequence diagram below, showing how the `add-expense` command is processed.

![add_expense.png](images/add_expense.png)

Why `FinTrack` was implemented this way:

- **Separation of Concerns**: The `FinTrack` class acts purely as a controller. It doesn't know how to parse data (`Parser`), how to store data (`FinanceManager`), or how to display information (`Ui`). This makes the code highly modular and easy to maintain.
- **Centralised Error Handling**: By wrapping the command execution in a `try-catch` block, the application is resilient. A malformed command (which throws an `IllegalArgumentException` from `Parser`) doesn't crash the program; it simply prints an error and allows the user to try again.
- **Simplicity**: A `switch` statement on the command word is the most direct and readable wait to implement a REPL for this set of commands.

### Ui Module (`Ui.java`)

#### Console Facade Overview

`Ui` (`src/main/java/seedu/fintrack/Ui.java`) is the single entry point for all console interaction in FinTrack. The class is intentionally static: it exposes command keywords, reads raw user input, and renders every message shown to the user without requiring an object to be instantiated. This keeps the rest of the application (parser, command executors, and model layer) free from I/O concerns while guaranteeing that the console state is mutated from a single place.

#### Command Token Registry

All canonical command phrases and parameter prefixes (`HELP_COMMAND`, `ADD_EXPENSE_COMMAND`, `AMOUNT_PREFIX`, and others) are defined as `public static final` constants at the top of the class. Centralising the tokens avoids string drift between the parser, help text, and automated tests. Any new user-visible command must be added here first, followed by updates to `printHelp()` so the usage documentation always reflects reality.

#### Input Loop Integration

`waitForInput()` owns the blocking read from `System.in` via a shared `Scanner`. The method prints a consistent `> ` prompt, trims whitespace, and returns an empty string when the user simply presses enter. If the input stream is closed or the scanner encounters an illegal state, the method logs the failure (`SEVERE`) and returns `EXIT_COMMAND`; this sentinel gives the caller a deterministic way to trigger a graceful shutdown without duplicating exception handling logic. Unexpected runtime exceptions are rethrown after being logged so they can be surfaced during development.

![img_1.png](images/img_1.png)

#### Output Formatters

- **Shared helpers**: `printHorizontalLine(int length)` is the only low-level formatter. It validates its argument, asserts the precondition when assertions are enabled, and writes the divider used by the list renderers.
- **Welcome and exit**: `printWelcome()` and `printExit()` wrap the banner text with INFO-level logs so session start and end are traceable in diagnostic logs.
- **Error surface**: `printError(String message)` prefixes every failure with `"Error: "` for consistent user feedback and mirrors the message to the logger at `WARNING` level. The duplication means log archives can be searched without relying on console captures.

#### Domain Object Renderers

All methods that print `Expense`, `Income`, or `ExpenseCategory` objects follow the same pattern: enforce non-null preconditions with `Objects.requireNonNull`, assert numeric invariants (`Double.isFinite`), and render values with explicit formatting. `printIncomeAdded(...)`, `printExpenseAdded(...)`, `printIncomeModified(...)`, and `printExpenseModified(...)` display the canonical summary after create or update operations. The deletion counterparts (`printIncomeDeleted(...)`, `printExpenseDeleted(...)`) warn if an invalid index slips through so upstream callers can be fixed during testing. Optional descriptions are only printed when non-blank, avoiding empty shells in the console output.

#### Balance and Budget Reporting

`printBalance(...)` centralises the three-line financial summary, keeping the numeric formatting consistent across commands. Budget-related methods highlight different phases: `printBudgetSet(...)` acknowledges successful configuration, `printBudgetExceededWarning(...)` emits an attention-grabbing banner when the spending threshold is crossed, and `printBudgets(...)` lists all configured budgets sorted by category using `Map.entrySet().stream().sorted(...)`. Sorting within the formatter decouples presentation order from the underlying storage implementation and keeps the console output digestible.

#### List Views

`printListOfIncomes(...)` and `printListOfExpenses(...)` render collections supplied by the model. Both methods iterate defensively: each entry is validated inside the loop, and malformed records are skipped with a `WARNING` log instead of aborting the entire render. Income entries are printed oldest first to match insertion order, while expenses are shown newest first to surface recent spending. Each row is wrapped with a 50-character horizontal divider to improve readability, and dates are standardised to `yyyy-MM-dd` via a local `DateTimeFormatter`.

![img.png](images/img.png)

#### Logging and Diagnostics

A dedicated `java.util.logging.Logger` instance records every significant branch at either INFO, FINE, WARNING, or SEVERE. Verbose (`FINE`) messages are emitted after successful renders, making it easy to cross-check console output with the log file when troubleshooting. Assertions complement logging by surfacing programmer errors (e.g., negative indices) during development builds without affecting production behaviour when assertions are disabled.

#### Extensibility Notes

When introducing new user flows:

- Add command keywords and prefixes alongside the existing constants so the parser and help text can reuse them.
- Implement a formatter that mirrors the defensive style (`Objects.requireNonNull`, assertions, and logging) before wiring it into higher-level commands.
- Keep console writing confined to `Ui`; other classes should pass domain objects or DTOs into these helpers instead of printing directly.
- Update or extend unit tests around `Ui` to lock in the new formatting, especially when the output participates in regression tests or scripted demos.

Because every method is static (most with package-private visibility), they can be invoked directly from tests within the same package without faking I/O streams. If a future feature requires richer presentation (e.g., table layout or different locales), the current structure allows the helper methods to be swapped for formatter objects while preserving the public surface area exposed to the rest of the application.

### Parser Module (`Parser.java`)

`Parser` (`src/main/java/seedu/fintrack/Parser.java`) is a stateless utility class (marked `final` with a `private` constructor) responsible for transforming raw user `String` input into structured validated data objects.

How the `Parser` component works:

1. The private `getValue(String args, String prefix)` method is the core of the parser. It works by:
   - Finding the start of a given prefix (e.g. `a/`).
   - Finding the start of the next known prefix (e.g. `c/`) using the `findNextPrefixIndex()` helper.
   - Extracting the substring between these two points as the value. This logic allows the user to provide arguments in any order (e.g. `c/food a/10` is the same as `a/10 c/food`).
2. Methods like `parseAddExpense(input)` orchestrate the parsing process:
   - The command word (e.g. `add-expense`) is stripped from the input string.
   - `getValue()` is called for each required argument (e.g. `a/`, `c/`, `d/`). If any return `null`, an `IllegalArgumentException` is thrown.
   - `getOptionalValue()` (a null-safe wrapper for `getValue()`) is called for optional arguments (e.g. `desc/`).
   - Type conversion and validation is performed on the extracted string values (e.g. `Double.parseDouble()`, `LocalDate.parse()`, `ExpenseCategory.parse()`).
   - If all validations pass, they construct and return the new data object (e.g. `new Expense(...)`). If any validation fails (e.g. `NumberFormatException`), it is caught and re-thrown as an `IllegalArgumentException` with a user-friendly message.
3. Methods like `parseDeleteExpense(input)` are simpler. The command word is simply stripped and the remaining string is parsed as a positive integer.

The internal logic for `parseAddExpense` is shown below:

![parser.png](images/parser.png)

Why `Parser` was implemented this way:

- **Single Responsibility Principle (SRP)**: The `Parser` class is a good example of SRP, as it only knows how to parse strings. It has no knowledge of `FinanceManager`, storage, or how the `Ui` works. This makes it independently testable and reusable.
- **Defensive Programming**: The `Parser` is the application's first line of defense against bad user input. It is designed to be extremely strict, throwing an `IllegalArgumentException` for any deviation from the expected format. This simplifies the rest of the application, as `FinTrack` and `FinanceManager` can trust that any object they receive from the `Parser` is valid.
- **Stateless Utility**: By making the class `final` with a `private` constructor and all `static` methods, we enforce that it's a stateless utility. There is no need to create an instance of a `Parser`, which simplifies the design.

Alternatives considered:

- **Positional Parsing**: An alternative design would be to use positional parsing (e.g. `add-expense 10 food 2025-10-22`). This was rejected as it's rigid and not user-friendly; the user must remember the exact order of arguments. The chosen prefix-based system (`a/`, `c/`) is more flexible.
- **Regex Parsing**: Another alternative was to use complex Regular Expressions (Regex) for each command. This was deemed harder to maintain and debug compared to the current prefix-scanning approach.

### FinanceManager Module (`FinanceManager.java`)

`FinanceManager` (`src/main/java/seedu/fintrack/FinanceManager.java`) is the central component that manages all financial data in FinTrack.  
It handles the core logic for adding, deleting, and retrieving transactions, tracking budgets, and calculating summaries such as total income, total expense, and overall balance.

#### How the `FinanceManager` component works:
1. **State management**  
   The class maintains three key data structures:
    - `IncomeList` — stores all incomes in reverse-chronological order
    - `ExpenseList` — stores all expenses in reverse-chronological order
    - `HashMap<ExpenseCategory, Double>` — stores budgets for each expense category  
      These are kept private to prevent external modification. Public “view” methods return unmodifiable copies of the data.

2. **Adding new records**
    - `addIncome(Income)` adds a new income to `IncomeList` after validation.
    - `addExpense(Expense)` adds a new expense and checks if it exceeds its category’s budget for the first time.  
      Both methods log the operation and assert data integrity before completing.

3. **Deleting and modifying records**
    - `deleteExpense(int)` and `deleteIncome(int)` remove entries by 1-based visible index (newest-first).
    - `modifyExpense(...)` and `modifyIncome(...)` replace old records by deleting and re-adding the updated one.  
      These operations throw clear exceptions if invalid indices are provided.

4. **Budgets**
    - `setBudget(ExpenseCategory, double)` sets or updates a budget for a given category.
    - `getBudgetsView()` returns an unmodifiable map of all budgets.
    - During each `addExpense`, the method compares the category’s total spending against its budget and signals when it is first exceeded.

5. **Calculations and summaries**
    - `getTotalIncome()`, `getTotalExpense()`, and `getBalance()` compute aggregate values for display.
    - `getExpenseByCategory()` and `getIncomeByCategory()` provide category-wise breakdowns for the summary commands.
    - Filtering by month is supported through `getIncomesViewForMonth(YearMonth)` and `getExpensesViewForMonth(YearMonth)`.

6. **Data export**  
   The method `exportToCSV(Path)` outputs all incomes, expenses, and a summary section into a CSV file.  
   It automatically creates directories if they do not exist and logs each step of the export process.

The sequence diagram below shows how `FinTrack` delegates an expense addition to `FinanceManager`, and how it performs budget checks and interacts with `ExpenseList`.
![FinanceManager_add-expense_DG.png](images/FinanceManager_add-expense_DG.png)

#### Why `FinanceManager` was implemented this way:
- **Encapsulation and safety** – `FinanceManager` hides internal data structures and only exposes read-only views, ensuring all state changes happen through validated methods.
- **Separation of concerns** – `FinanceManager` focuses solely on business logic. Input parsing and user interaction are handled by `Parser` and `Ui` respectively, reducing coupling.
- **Defensive programming** – Assertions, null checks, and exception handling prevent data corruption and simplify debugging.
- **Testability** – Because `FinanceManager` performs no console I/O, its methods can be easily tested using mock `Expense` and `Income` objects.


## Implementation

This section describes some noteworthy details of how certain features are implemented.

### Budget (`budget and list-budget`)

The budget feature allows us to set budgets for our expenses which lets us set a budget for a certain category. When we set a budget using `budget c/{ExpenseCategory} a/{amount}`, we get warned when any expense using `add-expense` we make exceeds our budget. Finally, we can see all the budgets we have set through `list-budget`.

Here is how `budget` and `list-budget` works:

1. In `FinTrack`, `Parser` first handles our input by recognising the budget function and then parsing the input into `ExpenseCategory` and budget amount.
2. To store all our budgets, we create a HashMap called `budgets` which has `ExpenseCategory` as the key and the budget amount as the value.
3. A function `setBudget` is then called to set the budget for the input category. Finally, `Ui` calls `printBudgetSet()` to indicate budget has been set.
4. Now, when `addExpense()` is called during `add-expense`, a boolean called `budgetExceeded` checks for if the added expense exceeds the budget set for that category. If so, a warning is printed by `printBudgetExceededWarning()` in `Ui`.
5. When `list-budget` is called, `Ui` prints a list of the budgets by calling `printBudgets` which receives a printable version of all the budgets which we get from `getBudgetsView()`.

Below is the sequence diagram of an instance of `budget` and `add-expense`:
![budget.png](images/budget.png)

#### Design Considerations:

- We considered alternative ways to hold our budgets such as arrays, however, for the sake of readability and simplicity, we went with a HashMap

### Summary (`summary-expense and summary-income`)

The summary feature comes in two forms:

- `summary-expense`: gives a brief summary of overall expense
- `summary-income`: gives a brief summary of overall income

A summary consists of the following details:

- **Overall expense/income**
- **Breakdown by category**
- **Top Category**

Here is how `summary-expense` works:

1. We first consider what was needed in our summary. The main things we needed
   were total expenditure, a breakdown of expenditure and the top category in expenditure
2. To get total expense, we can get total expense from calling `getTotalExpense()` from `FinanceManager`.
3. To see how much the user has spent on each category, a function in `FinanceManager` called `getExpenseByCategory()` is implemented to return a HashMap which has `ExpenseCategory` as the key and the accumulated amount of that category as the value.
4. To create this hashmap, `getExpenseByCategory()` loops through all expenses in the `expenses` list and adds up the total amount for each `ExpenseCategory`.
5. `totalExpense` and `expenseByCategory` is then fed into a function in `Ui` called `printSummaryExpense` to print the summary.

Below is a sequence diagram to illustrate how summary-expense works:
![summary_expense.png](images/summary_expense.png)

`summary-income` is implemented in a similar way.

#### Design Considerations

- One thing we considered was how to implement this as simply as possible. While we recognise that the current implementation has a data-hungry implementation in a HashMap, it was also the simplest solution.
- This solution also helps to reduce coupling and improve testing of the implementation.

### Export (`export`)
The export feature allows users to export all their financial data (incomes and expenses) to a CSV file for backup, analysis in spreadsheet applications, or record-keeping purposes.

Here is how `export` works:
1. In `FinTrack`, `Parser` handles the input by recognising the export command and parsing the file path using `parseExport(input)`.
2. A new `CsvStorage` instance is created to handle the file I/O operations (following the Single Responsibility Principle by keeping storage concerns separate from business logic).
3. `FinTrack` retrieves all necessary data from `FinanceManager`:
   - `getIncomesView()` - Returns an unmodifiable list of all incomes
   - `getExpensesView()` - Returns an unmodifiable list of all expenses
   - `getTotalIncome()`, `getTotalExpense()`, `getBalance()` - Summary statistics
4. The `CsvStorage.export()` method writes the data to a CSV file in proper CSV format:
   - Single header row: `Type,Date,Amount,Category,Description`
   - All incomes and expenses in one unified table with a `Type` column distinguishing between "INCOME" and "EXPENSE"
   - Summary section at the end with total income, total expenses, and balance
5. If successful, `Ui` calls `printExportSuccess()` to confirm the export. Any errors (IOException, SecurityException, IllegalArgumentException) are caught and displayed to the user via `printError()`.

Below is a sequence diagram to illustrate how the export command works:
![export.png](images/export.png)

#### Design Considerations:
**Separation of Concerns via Storage Layer:**
- **Why we created a separate Storage layer**: Initially, export functionality was in `FinanceManager`, which violated the Single Responsibility Principle. `FinanceManager` should manage financial data and business logic, not handle file I/O.
- **Alternative considered**: Keeping `exportToCSV()` in `FinanceManager` - This was rejected because:
  - It mixed business logic with persistence concerns
  - It would make `FinanceManager` harder to test
  - Adding new export formats (JSON, XML) would bloat the class
- **Why this approach is better**: 
  - `Storage` interface allows for multiple implementations (CSV, JSON, etc.)
  - Follows the same stateless pattern as `Parser` and `Ui`
  - Makes the code more maintainable and testable
  - Sets foundation for future features like auto-save and data import

**CSV Format Choice:**
- **Single-table format with Type column**: We chose to have all transactions in one table with a "Type" column (INCOME/EXPENSE) rather than separate sections
- **Alternative considered**: Separate INCOMES and EXPENSES sections - This was rejected because:
  - Not standard CSV format
  - Harder to import into spreadsheet applications
  - Cannot be easily sorted or filtered as a unified dataset
- **Why single-table is better**: Proper CSV format that works seamlessly with Excel, Google Sheets, and data analysis tools
### Modify (`modify-expense and modify-income`)

The modify feature allows users to update existing financial entries without having to delete and manually re-add them. This feature comes in two forms:

- `modify-expense`: Updates an existing expense entry at a specified index
- `modify-income`: Updates an existing income entry at a specified index

Both commands work by replacing the old entry with new data while maintaining the chronological order of the list.

#### How `modify-expense` works:

1. In `FinTrack`, `Parser` handles the input by recognising the `modify-expense` command.
2. `Parser.parseModifyExpense(input)` extracts two key pieces of information:
   - The **index** (1-based position in the expense list)
   - The **new expense data** (amount, category, date, optional description)
3. The parser validates the index is a positive integer, then reuses the existing `parseAddExpense()` logic to parse the remaining parameters (amount, category, date, description).
4. The parser returns a `Map.Entry<Integer, Expense>` containing both the index and the new expense object.
5. `FinTrack` calls `fm.modifyExpense(index, newExpense)` which:
   - Calls `deleteExpense(index)` to remove the old expense and stores it temporarily
   - Calls `addExpense(newExpense)` to add the new expense (which maintains reverse chronological ordering)
   - If adding the new expense fails for any reason, the old expense is restored to maintain data integrity
   - Returns a boolean indicating if the modification caused the category budget to be exceeded
6. `Ui.printExpenseModified()` confirms the successful modification.
7. If the budget was exceeded, additional warnings are displayed via `printBudgetExceededWarning()`.

Below is a sequence diagram illustrating the `modify-expense` flow:

![modify_expense.png](images/modify_expense.png)

#### How `modify-income` works:

1. In `FinTrack`, `Parser` handles the input by recognising the `modify-income` command.
2. `Parser.parseModifyIncome(input)` extracts two key pieces of information:
   - The **index** (1-based position in the income list)
   - The **new income data** (amount, category, date, optional description)
3. The parser validates the index is a positive integer, then reuses the existing `parseAddIncome()` logic to parse the remaining parameters.
4. The parser returns a `Map.Entry<Integer, Income>` containing both the index and the new income object.
5. `FinTrack` calls `fm.modifyIncome(index, newIncome)` which:
   - Calls `deleteIncome(index)` to remove the old income and stores it temporarily
   - Calls `addIncome(newIncome)` to add the new income (which maintains reverse chronological ordering)
   - If adding the new income fails for any reason, the old income is restored to maintain data integrity
6. `Ui.printIncomeModified()` confirms the successful modification.

Below is a sequence diagram illustrating the `modify-income` flow:

![modify_income.png](images/modify_income.png)

#### Design Considerations:

**Delete-and-Add Approach:**

- **Why this approach**: The modify operations are implemented as a delete followed by an add operation, rather than direct in-place modification.
- **Advantages**:
  - **Code Reuse**: Leverages existing, well-tested `deleteExpense()`/`deleteIncome()` and `addExpense()`/`addIncome()` methods
  - **Consistency**: Ensures that modified entries are automatically re-sorted into the correct chronological position
  - **Budget Checking**: For expenses, the add operation automatically checks budget constraints
  - **Validation**: All validation logic from the add operations is automatically applied
- **Alternatives considered**:
  - Direct in-place modification - This was rejected because:
    - Would require duplicating validation logic
    - Would need manual re-sorting if the date changes
    - Would complicate budget checking logic
    - Increases code complexity and potential for bugs

**Rollback on Failure:**

- **Transaction-like behavior**: If adding the new entry fails, the old entry is restored from the temporary variable
- **Why this is important**: Prevents data loss if the new entry has invalid data that passes parser validation but fails business logic validation
- **User experience**: Users never lose their original data due to a failed modification attempt

**Parser Reuse:**

- **Smart parsing**: The modify parsers extract the index first, then prepend the appropriate command (`add-expense` or `add-income`) to the remaining arguments and call the existing add parsers
- **Benefits**:
  - Eliminates code duplication
  - Ensures modify commands support exactly the same format as add commands
  - Any improvements to add parsing automatically benefit modify parsing

**Index Validation:**

- Both modify methods provide clear, specific error messages:
  - "Cannot modify expense/income: The list is empty" when the list has no items
  - "Index out of range. Valid range: 1 to N" when the index is invalid
  - Helps users understand exactly what went wrong

## Product scope

### Target user profile

{Describe the target user profile}

### Value proposition

{Describe the value proposition: what problem does it solve?}

## User Stories

| Priority | As a ... | I want to ... | So that I can ... |
| -------- | -------- | ------------- | ----------------- |
| High | newly enrolled NUS computing student | run a single `help` command that lists every available command with concrete examples | orient myself quickly between lectures and lab sessions |
| High | allowance-conscious engineering undergrad | capture each expense with amount, category, date, and optional description in one command | keep an accurate record of tuition fees, accommodation, and component purchases |
| High | NUS intern juggling stipends | log recurring income entries with category tags and posting dates | align internship stipends, scholarships, and allowances against upcoming costs |
| High | student project lead | define monthly budgets per category and receive warnings when adding expenses that exceed them | keep capstone or design project spending within reimbursement limits |
| Medium | analytical NUS student | list and filter expenses/incomes by category or month directly from the CLI | review spending by modules, labs, or campus activities without exporting data |
| Medium | meticulous student | modify previously recorded entries by index using the same prefixes as the add command | correct mistakes before submitting receipts for reimbursement |
| Medium | collaborative project teammate | export my financial data to a CSV file at a user-specified path | share spending breakdowns with teammates or supervisors during project reviews |
| Low | exchange-aspiring student | display a summary of income versus expenses over a configurable time range | plan savings targets for SEP or overseas internships |
| Low | busy student leader | exit the application safely with `bye` once I'm done recording transactions | wrap up quickly between lectures without leaving the CLI session hanging |

## Non-Functional Requirements

### Performance
- **Responsiveness**: All commands must provide feedback to the user (either a result or an error message) in under 1 second, assuming a data set of up to 10,000 entries (incomes + expenses). This ensures the CLI application feels instantaneous.
- **Resource Consumption**: The application should run efficiently on a typical personal computer, with minimal CPU and memory footprint during idle state (waiting for input).

### Usability
- **Error Feedback**: The application must never crash on invalid user input. All user errors (e.g., bad command syntax, invalid dates, non-numeric amounts) must be caught and reported with a clear, actionable error message via `Ui.printError()`.
- **Learnability**: The command syntax must be consistent. All commands that take arguments must use the prefix-based system (e.g., `a/`, `c/`, `d/`) to minimise the user's cognitive load.
- **Guidance**: A comprehensive `help` command must be available to list all available commands and their syntax.

### Maintainability
- **Separation of Concerns (SoC)**: The architecture must strictly enforce SoC.
    - `Ui`: Handles all console input and output. No business logic.
    - `Parser`: Handles all string parsing and user-input validation. No business logic or I/O.
    - `FinanceManager`: Contains all business logic and state management. No parsing or direct I/O.
    - `model` (e.g., `Income`, `Expense`): Plain data objects with constructors for validation.
- **Testability**: All business logic (`FinanceManager`) and parsing logic (`Parser`) must be decoupled from the `Ui`, allowing them to be unit-tested without mocking `System.in` or `System.out`.
- **Extensibility**: Adding a new command must follow a clear pattern (adding constants to `Ui`, a case to `FinTrack`, and methods to `Parser`, `FinanceManager`, and `Ui`) without requiring modifications to existing, unrelated commands.

### Portability
- **Platform Independence**: The application must be runnable on any operating system (Windows, macOS, Linux) that has a compatible Java Runtime Environment (JRE) installed.
- **File System**: The `export` feature must handle different file system path conventions (e.g., ~ for home directory) and gracefully report file permission or path-not-found errors.

### Security
- **Local Data**: All user data is stored in-memory during runtime and is never transmitted over any network.
- **File Access**: The application must only interact with the file system when explicitly requested by the user (e.g., via the `export` command) and must handle SecurityException if it lacks permission to write to a specified path.

## Glossary

- _glossary item_ - Definition

## Instructions for manual testing

{Give instructions on how to do a manual product testing e.g., how to load sample data to be used for testing}
