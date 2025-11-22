

1) Header includes and macros
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <time.h>


stdio.h — input/output (printf, scanf, FILE, fopen, etc.).
string.h — string helpers (strcpy, strlen, strcasecmp, strcspn).
stdlib.h — general utilities (exit, system).
time.h — time functions (time, ctime).

#define MAX_ITEMS 100
#define MAX_NAME_LENGTH 50
#define MAX_CATEGORY 30
#define MENU_FILE "menu.txt"
#define SALES_FILE "sales.txt"
#define BILL_FILE "bill.txt"


Constants used throughout: maximum sizes and filenames used for persistence.

2) MenuItem struct and globals
typedef struct {
    int id;
    char name[MAX_NAME_LENGTH];
    char category[MAX_CATEGORY];
    float price;
    int quantity;
} MenuItem;

MenuItem menu[MAX_ITEMS];
int itemCount = 0;


MenuItem holds one menu entry: unique id, name, category, price, available quantity.

menu is an in-memory array storing items; itemCount tracks how many are loaded/stored.

3) Function prototypes

They declare all functions used later so the compiler knows them before definitions. (Good C style for ordering.)

4) main() — program entry & menu loop
int main() {
    system("color 3F"); // Blue UI with white text
    adminLogin();
    loadMenuFromFile();

    int choice;
    printf("=== RESTAURANT BILLING SYSTEM (₹) ===\n");

    do {
        // prints main menu and reads choice
        scanf("%d", &choice);
        switch(choice) {
            case 1: displayMenu(); break;
            ...
            case 9: saveMenuToFile(); printf("Saved & Exit!\n"); break;
            default: printf("Invalid choice. Try again!\n");
        }
    } while(choice != 9);

    return 0;
}


system("color 3F"): Windows console color code (nonportable). 3F means background blue, bright white text.

Calls adminLogin() first — user must enter correct password or the program exits.

loadMenuFromFile() reads previously saved menu into memory (if exists).

Then a loop shows options and calls functions based on user choice. Option 9 saves and exits.

5) adminLogin()
void adminLogin() {
    char pass[20];
    printf("Enter Admin Password: ");
    scanf("%s", pass);
    if(strcmp(pass, "admin123") != 0) {
        printf("Incorrect Password! Access Denied.\n");
        exit(0);
    }
    printf("Access Granted!\n");
}


Simple password check using strcmp. If password mismatch, program exit(0).

Caveats: password stored in source, scanf("%s") can overflow if input longer than buffer — better use fgets + trim and compare, and never store plain passwords in code.

6) displayMenu() — show items for customers
void displayMenu() { ... }


Prints headers and then loops menu[0..itemCount-1], printing ID, name, price and category.

Uses ₹ symbol (UTF-8) to show Indian Rupee; console must support it.

7) displayAllItems() — admin view including quantities

Similar to displayMenu() but includes quantity. Useful to manage stock.

8) addItem() — add new menu item
void addItem() {
    if(itemCount >= MAX_ITEMS) { ... }
    MenuItem newItem;
    scanf("%d", &newItem.id);
    if(findItemById(newItem.id) != -1) { ... }
    getchar();
    fgets(newItem.name, MAX_NAME_LENGTH, stdin); // read name
    fgets(newItem.category, MAX_CATEGORY, stdin);
    scanf("%f", &newItem.price);
    scanf("%d", &newItem.quantity);
    menu[itemCount] = newItem;
    itemCount++;
    saveMenuToFile();
}


Reads ID, name, category, price, quantity from user and appends to menu.

Calls findItemById() to prevent duplicate IDs.

Immediately calls saveMenuToFile() so additions persist.

I/O notes: Uses getchar(); fgets(...); to handle newline after numeric input — mixing scanf and fgets requires care (this code mostly handles it but still fragile).

Validation: checks if menu array is full and duplicates, but does not validate negative prices, negative quantities, or invalid text lengths.

9) deleteItem() — remove by ID

Reads an ID, finds index via findItemById, shifts subsequent elements left to overwrite deleted item, decrements itemCount, saves file.

10) updateItem() — modify existing item

Finds item by ID, offers:

New name: reads a line, if length > 1 then replaces.

New price: if entered > 0 then replaces.

New quantity: if >= 0 then replaces.

Saves to file.

11) searchItem() — search by exact name

Reads search string, uses strcasecmp for case-insensitive exact match with the stored name.

If found prints name, price, category.

Limitation: exact match only; no substring or partial search.

12) Sorting functions: sortMenu(), sortByPrice(), sortByName()

sortMenu() asks for choice and calls relevant sorter.

sortByPrice() and sortByName() implement simple bubble-like O(n²) sorts swapping MenuItem structs.

strcasecmp used in sortByName() for case-insensitive comparison.

13) lowStockCheck(int index)
void lowStockCheck(int index) {
    if(menu[index].quantity < 5)
        printf("⚠ Low Stock Warning: Only %d left!\n", menu[index].quantity);
}


Called when items are sold to warn if quantity drops below 5.

14) generateBill() — main billing flow

This is one of the longer functions; key steps:

Check itemCount and return if none.

Ask Dine-In/Takeaway (stored in orderType but not used later — possible extension point).

Calls displayMenu() to show items.

Loop to build the order:

Prompt for item ID.

Validate ID using findItemById().

Prompt for quantity; check qty > 0 && qty <= menu[index].quantity.

Decrease menu[index].quantity immediately (stock updated).

Store selected index in orderItems[], orderQty[].

Call lowStockCheck(index).

Ask add more items (read char).

After ordering, print bill table: item, price, qty, total for each line and accumulate grandTotal.

Ask for percentage discount and percentage gstRate.

Compute:

discountAmt = grandTotal * discount / 100

gstAmt = grandTotal * gstRate / 100

finalTotal = grandTotal - discountAmt + gstAmt

Print subtotal/discount/gst/final total.

Append sale to SALES_FILE:

FILE *sales = fopen(SALES_FILE, "a");
time_t t; time(&t);
fprintf(sales, "Bill: ₹%.2f Date: %s", finalTotal, ctime(&t));
fclose(sales);


This writes a simple line with final total and a human-readable date/time (ctime includes newline at end).

Calls saveMenuToFile() to persist updated quantities.

Prints a thank-you.

Notes & caveats:

orderType is collected but unused — you might want to use it to add a takeaway packing charge or different tax.

No per-item discounts, no rounding adjustment or formatting of GST vs tax-inclusive/exclusive rules.

The sales file only records finalTotal and date — does not save itemized bill lines; useful if you want a more detailed record to audit later.

Payment flow (cash/card) not implemented.

15) findItemById(int id)

Linear search returning array index or -1 if not found.

16) saveMenuToFile() and loadMenuFromFile()
void saveMenuToFile() {
    FILE *file = fopen(MENU_FILE, "w");
    fprintf(file, "%d\n", itemCount);
    for(...) fprintf(file, "%d\n%s\n%s\n%.2f\n%d\n", ...);
    fclose(file);
}


Writes itemCount first, then for each item writes: id, name (newline), category (newline), price, quantity — each on its own line.

Format example for two items would look like:

2
101
Paneer Butter Masala
Main Course
180.00
10
102
Veg Biryani
Rice
120.00
5


loadMenuFromFile() reads file in same order and populates menu[].

Robustness issues:

saveMenuToFile() does not check whether fopen returned NULL — writing can fail and code will crash when fprintf called on a NULL pointer. Always test if(file == NULL).

loadMenuFromFile() does FILE *file = fopen(MENU_FILE, "r"); if (file == NULL) return; — good.

loadMenuFromFile() uses fscanf and fgets mixing; this works because they handle newlines carefully but is brittle. Also no bounds-check for fscanf return values.

17) I/O mixing and common pitfalls

Mixing scanf with %d and then fgets requires getchar() to remove newline. The code uses getchar() before fgets in several places — that mostly works but is fragile in real usage (extra whitespace/newlines, stray chars). Prefer consistent input handling (read entire line with fgets and parse with sscanf or strtol).

scanf("%s", pass) in adminLogin() can overflow pass[20] if user types a very long string. Use fgets with buffer size, then trim newline.

18) Security & UX suggestions (improvements)

Password: don’t hardcode; load from config or prompt and compare hashed values. At minimum, avoid plain text in the source.

File error handling: check fopen success and check fprintf/fscanf return values.

Atomic saves: write to temporary file and rename to avoid corrupt menu.txt if interrupted.

Sales log: record itemized bills (IDs, qty, line totals) and an invoice ID for easier auditing.

Input validation: ensure price ≥ 0, quantity ≥ 0, ID uniqueness, name/category length checks.

Concurrency: if multiple terminals access same files, locking is required (not covered).

Internationalization: be careful with ₹ in terminals that may not render it.

User roles: separate admin/customer menus; currently adminLogin is required before all functions. You might let non-admin view/display menu without password.

Formatting: round money values properly and format totals to 2 decimal places (code already prints with %.2f).

Use safer functions: fgets + sscanf, strncpy, check lengths to avoid buffer overflows.

19) Minor bugs / improvements to fix quickly

saveMenuToFile() — add if(file == NULL) { perror("Could not save menu"); return; }

generateBill() — orderType currently unused — either remove or use it.

searchItem() uses strcasecmp to compare names — if you want substring search use strcasestr or implement your own.

BILL_FILE macro exists but is unused (you currently append bills to SALES_FILE). Remove BILL_FILE or use it to save each bill as a separate file / generate invoice file.

20) Example flow (what files look like)

menu.txt (as written by saveMenuToFile()):

1
101
Tea
Beverages
20.00
50


sales.txt (append lines like):

Bill: ₹150.00 Date: Sat Nov 22 18:40:00 2025

21) Summary — what the program does

Manages a menu of items in memory and saves it to menu.txt.

Provides admin-only CRUD on menu items (add, delete, update).

Displays menu for customers, generates bills with discount and GST, updates stock quantities, warns low stock, and logs sale totals with timestamp.

Simple persistent storage through plain text files.
