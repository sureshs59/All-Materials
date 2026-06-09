# Java String Methods: Complete Guide with Examples

**Author:** Technical Reference  
**Date:** June 2026  
**Purpose:** Comprehensive guide for Java String operations with real-world use cases

---

## Table of Contents

1. [String Basics](#string-basics)
2. [String Length & Character Access](#string-length--character-access)
3. [String Comparison Methods](#string-comparison-methods)
4. [String Searching Methods](#string-searching-methods)
5. [String Manipulation Methods](#string-manipulation-methods)
6. [String Transformation Methods](#string-transformation-methods)
7. [String Formatting Methods](#string-formatting-methods)
8. [Advanced String Operations](#advanced-string-operations)
9. [String Immutability & Performance](#string-immutability--performance)
10. [Real-World Examples](#real-world-examples)

---

## String Basics

### Creating Strings

```java
// Method 1: String literal (most common, stored in String pool)
String name = "John Doe";

// Method 2: Using new keyword (creates new object in heap)
String name2 = new String("John Doe");

// Method 3: From character array
char[] chars = {'J', 'o', 'h', 'n'};
String name3 = new String(chars);

// Method 4: From bytes
byte[] bytes = "John".getBytes();
String name4 = new String(bytes);

// String is IMMUTABLE - cannot be changed after creation
String original = "Hello";
String modified = original.concat(" World"); // Creates new String object
System.out.println(original); // Still "Hello"
System.out.println(modified); // "Hello World"
```

**Key Concept:** Strings are immutable. Every operation returns a new String object.

---

## String Length & Character Access

### 1. `length()` - Get String Length

```java
String text = "Hello World";
int length = text.length(); // Returns 11

/**
 * Real-world use case:
 * - Validating input length (passwords, usernames)
 * - Checking if string is empty
 * - Looping through characters
 */

// Example: Validate password length
public boolean isValidPassword(String password) {
    return password.length() >= 8 && password.length() <= 20;
}

// Example: Check if empty
if (text.length() == 0) {
    System.out.println("String is empty");
}

// Example: Loop through characters
for (int i = 0; i < text.length(); i++) {
    System.out.print(text.charAt(i) + " ");
}
// Output: H e l l o   W o r l d
```

**When to use:**
- Input validation
- Buffer size checking
- Loop iterations

---

### 2. `charAt(int index)` - Get Character at Index

```java
String text = "Hello";
char firstChar = text.charAt(0);        // 'H'
char lastChar = text.charAt(text.length() - 1); // 'o'
char middleChar = text.charAt(2);       // 'l'

/**
 * Real-world use case:
 * - Accessing specific characters
 * - Building strings character by character
 * - Character-level processing
 */

// Example: Check if string starts with uppercase
public boolean startsWithUpperCase(String text) {
    if (text.length() == 0) return false;
    return Character.isUpperCase(text.charAt(0));
}

// Example: Reverse a string
public String reverseString(String text) {
    StringBuilder reversed = new StringBuilder();
    for (int i = text.length() - 1; i >= 0; i--) {
        reversed.append(text.charAt(i));
    }
    return reversed.toString();
}
// reverseString("Hello") → "olleH"

// Example: Count vowels
public int countVowels(String text) {
    int count = 0;
    for (int i = 0; i < text.length(); i++) {
        char c = text.charAt(i);
        if ("aeiouAEIOU".contains(String.valueOf(c))) {
            count++;
        }
    }
    return count;
}
// countVowels("Hello World") → 3
```

**When to use:**
- Character-level validation
- String reversal
- Vowel/consonant counting
- Character iteration

---

### 3. `isEmpty()` - Check if String is Empty

```java
String text1 = "";
String text2 = "Hello";

boolean empty1 = text1.isEmpty(); // true
boolean empty2 = text2.isEmpty(); // false

/**
 * Real-world use case:
 * - Form validation
 * - Null-safe checks
 * - Conditional processing
 */

// Example: Validate form input
public boolean isValidEmail(String email) {
    return !email.isEmpty() && email.contains("@");
}

// Example: Initialize with default
String userInput = getUserInput();
String displayText = userInput.isEmpty() ? "No input provided" : userInput;

// Note: Use isBlank() for strings with only whitespace (Java 11+)
String whitespace = "   ";
whitespace.isEmpty(); // false (has spaces)
whitespace.isBlank(); // true (only whitespace)
```

**When to use:**
- Form validation
- Null checks
- Conditional initialization

---

## String Comparison Methods

### 4. `equals(Object obj)` - Case-Sensitive Comparison

```java
String text1 = "Hello";
String text2 = "Hello";
String text3 = "hello";

boolean result1 = text1.equals(text2);      // true
boolean result2 = text1.equals(text3);      // false (different case)

/**
 * Real-world use case:
 * - Exact string matching
 * - Password verification
 * - Authentication
 */

// Example: Verify password (case-sensitive)
public boolean verifyPassword(String inputPassword, String storedPassword) {
    return inputPassword.equals(storedPassword);
}

// Example: Permission checking
public boolean hasPermission(String userRole, String requiredRole) {
    return userRole.equals(requiredRole);
}

// Example: State comparison
public boolean isAdminUser(String role) {
    return role.equals("ADMIN");
}

// Good practice: Avoid NullPointerException
String userRole = getUserRole(); // might be null
if (userRole != null && userRole.equals("ADMIN")) {
    // Safe to use
}

// Better: Use Objects.equals() (null-safe)
Objects.equals(userRole, "ADMIN"); // Returns false if userRole is null
```

**When to use:**
- Exact matching required
- Case sensitivity matters
- Authentication/authorization

---

### 5. `equalsIgnoreCase(String anotherString)` - Case-Insensitive Comparison

```java
String text1 = "Hello";
String text2 = "HELLO";
String text3 = "hello";

boolean result = text1.equalsIgnoreCase(text2); // true
boolean result2 = text1.equalsIgnoreCase(text3); // true

/**
 * Real-world use case:
 * - Case-insensitive matching
 * - Language/locale handling
 * - User input validation
 */

// Example: Case-insensitive command processing
public void processCommand(String command) {
    if (command.equalsIgnoreCase("QUIT")) {
        System.exit(0);
    } else if (command.equalsIgnoreCase("HELP")) {
        showHelp();
    }
}

// Example: File extension checking
public boolean isImageFile(String filename) {
    return filename.endsWith(".jpg") ||
           filename.endsWith(".jpeg") ||
           filename.endsWith(".png") ||
           filename.endsWith(".gif");
    // Better: Use equalsIgnoreCase
}

public boolean isImageFileBetter(String filename) {
    String[] extensions = {".jpg", ".jpeg", ".png", ".gif"};
    for (String ext : extensions) {
        if (filename.toLowerCase().endsWith(ext)) {
            return true;
        }
    }
    return false;
}

// Example: Email comparison (emails are case-insensitive)
public boolean isRegisteredEmail(String email) {
    String storedEmail = "John.Doe@example.com";
    return storedEmail.equalsIgnoreCase(email);
}
```

**When to use:**
- User input validation
- File extensions
- Email addresses
- Case-insensitive matching

---

### 6. `compareTo(String anotherString)` - Lexicographic Comparison

```java
String text1 = "Apple";
String text2 = "Banana";
String text3 = "Apple";

int result1 = text1.compareTo(text2); // Negative (Apple < Banana)
int result2 = text1.compareTo(text3); // 0 (equal)
int result3 = text2.compareTo(text1); // Positive (Banana > Apple)

// Returns:
// - Negative: text1 < text2
// - 0: text1 == text2
// - Positive: text1 > text2

/**
 * Real-world use case:
 * - Sorting strings
 * - Version comparison
 * - Lexicographic ordering
 */

// Example: Sorting user names
List<String> names = Arrays.asList("Zoe", "Alice", "Bob", "Charlie");
Collections.sort(names); // Uses compareTo() internally
// Result: [Alice, Bob, Charlie, Zoe]

// Example: Custom sorting with compareTo
List<String> users = Arrays.asList("zoe", "alice", "bob");
Collections.sort(users, String::compareTo);

// Example: Version comparison (simple)
public int compareVersions(String version1, String version2) {
    return version1.compareTo(version2);
}
// Note: For complex version comparison, use proper version classes

// Example: Binary search (requires sorted array)
String[] fruits = {"Apple", "Banana", "Cherry", "Date"};
Arrays.sort(fruits);
int index = Arrays.binarySearch(fruits, "Cherry"); // Works with compareTo

// Example: Case-insensitive comparison
public int compareIgnoreCase(String text1, String text2) {
    return text1.compareToIgnoreCase(text2);
}
```

**When to use:**
- Sorting arrays/lists
- Binary search
- Determining order
- Version comparison

---

## String Searching Methods

### 7. `indexOf(String substring)` - Find First Occurrence

```java
String text = "Hello World, Hello Universe";

int index1 = text.indexOf("Hello");      // 0 (first occurrence)
int index2 = text.indexOf("World");      // 6
int index3 = text.indexOf("xyz");        // -1 (not found)
int index4 = text.indexOf("o");          // 4 (first 'o')

// indexOf with starting position
int index5 = text.indexOf("Hello", 5);   // 13 (second Hello, starting from position 5)

/**
 * Real-world use case:
 * - Finding substrings
 * - Email validation
 * - File path parsing
 * - Text processing
 */

// Example: Extract domain from email
public String extractDomain(String email) {
    int atIndex = email.indexOf("@");
    if (atIndex == -1) {
        return ""; // Invalid email
    }
    return email.substring(atIndex + 1);
}
// extractDomain("john@example.com") → "example.com"

// Example: Check if email is valid
public boolean isValidEmail(String email) {
    int atIndex = email.indexOf("@");
    int dotIndex = email.indexOf(".");
    return atIndex > 0 && dotIndex > atIndex;
}

// Example: Find first occurrence and extract
public String getProtocol(String url) {
    int colonIndex = url.indexOf(":");
    return colonIndex == -1 ? "" : url.substring(0, colonIndex);
}
// getProtocol("https://example.com") → "https"

// Example: Find all occurrences
public List<Integer> findAllOccurrences(String text, String substring) {
    List<Integer> positions = new ArrayList<>();
    int index = 0;
    while ((index = text.indexOf(substring, index)) != -1) {
        positions.add(index);
        index += substring.length();
    }
    return positions;
}
// findAllOccurrences("Hello Hello Hello", "Hello") → [0, 6, 12]
```

**When to use:**
- Parsing strings
- Email extraction
- URL parsing
- Finding substrings

---

### 8. `lastIndexOf(String substring)` - Find Last Occurrence

```java
String text = "Hello World, Hello Universe";

int lastIndex1 = text.lastIndexOf("Hello");    // 13 (last occurrence)
int lastIndex2 = text.lastIndexOf("o");        // 26 (last 'o')
int lastIndex3 = text.lastIndexOf("xyz");      // -1 (not found)

/**
 * Real-world use case:
 * - File path parsing
 * - Getting file extensions
 * - Extracting filename
 */

// Example: Extract file extension
public String getFileExtension(String filename) {
    int lastDotIndex = filename.lastIndexOf(".");
    if (lastDotIndex == -1) {
        return ""; // No extension
    }
    return filename.substring(lastDotIndex + 1);
}
// getFileExtension("document.pdf") → "pdf"
// getFileExtension("archive.tar.gz") → "gz"

// Example: Extract filename from path
public String getFilename(String filePath) {
    int lastSlashIndex = filePath.lastIndexOf("/");
    return filePath.substring(lastSlashIndex + 1);
}
// getFilename("/home/user/documents/file.txt") → "file.txt"

// Example: Get file name without extension
public String getFileNameWithoutExtension(String filename) {
    int lastDotIndex = filename.lastIndexOf(".");
    if (lastDotIndex == -1) {
        return filename;
    }
    return filename.substring(0, lastDotIndex);
}
// getFileNameWithoutExtension("document.pdf") → "document"
```

**When to use:**
- File operations
- Path parsing
- Extension extraction
- Finding last occurrence

---

### 9. `contains(CharSequence sequence)` - Check if Contains Substring

```java
String text = "Hello World";

boolean contains1 = text.contains("World");     // true
boolean contains2 = text.contains("xyz");       // false
boolean contains3 = text.contains("o W");       // true
boolean contains4 = text.contains("");          // true (empty string)

/**
 * Real-world use case:
 * - Substring checking
 * - Input validation
 * - String filtering
 */

// Example: Filter adult content
public boolean containsProfanity(String text) {
    String[] badWords = {"badword1", "badword2", "badword3"};
    for (String word : badWords) {
        if (text.toLowerCase().contains(word)) {
            return true;
        }
    }
    return false;
}

// Example: Check if URL is valid
public boolean isValidWebsite(String url) {
    return url.contains("http://") || url.contains("https://");
}

// Example: Email domain validation
public boolean isCorpEmail(String email) {
    return email.contains("@company.com");
}

// Example: HTML parsing
public boolean hasHtmlTags(String text) {
    return text.contains("<") && text.contains(">");
}

// Example: String filtering
List<String> emails = Arrays.asList(
    "john@gmail.com",
    "alice@company.com",
    "bob@yahoo.com"
);
// Filter company emails
List<String> companyEmails = emails.stream()
    .filter(email -> email.contains("@company.com"))
    .collect(Collectors.toList());
```

**When to use:**
- Content filtering
- Validation
- String checking
- Data filtering

---

### 10. `startsWith(String prefix)` - Check if Starts With

```java
String text = "Hello World";

boolean starts1 = text.startsWith("Hello");     // true
boolean starts2 = text.startsWith("World");     // false
boolean starts3 = text.startsWith("H");         // true

// startsWith with offset
boolean starts4 = text.startsWith("World", 6);  // true (from position 6)

/**
 * Real-world use case:
 * - File validation
 * - URL parsing
 * - Protocol checking
 * - Version checking
 */

// Example: Check file type
public boolean isPythonFile(String filename) {
    return filename.endsWith(".py");
}

public boolean isJavaFile(String filename) {
    return filename.endsWith(".java");
}

// Example: URL validation
public boolean isSecureUrl(String url) {
    return url.startsWith("https://");
}

// Example: Log level filtering
public void processLog(String logLine) {
    if (logLine.startsWith("[ERROR]")) {
        handleError(logLine);
    } else if (logLine.startsWith("[WARN]")) {
        handleWarning(logLine);
    } else if (logLine.startsWith("[INFO]")) {
        handleInfo(logLine);
    }
}

// Example: Version compatibility check
public boolean isCompatibleVersion(String currentVersion) {
    return currentVersion.startsWith("2.0") || 
           currentVersion.startsWith("2.1");
}

// Example: Parse phone number by country code
public String getCountry(String phoneNumber) {
    if (phoneNumber.startsWith("+1")) {
        return "USA";
    } else if (phoneNumber.startsWith("+44")) {
        return "UK";
    } else if (phoneNumber.startsWith("+91")) {
        return "India";
    }
    return "Unknown";
}
```

**When to use:**
- Protocol checking
- File type validation
- Log parsing
- Prefix validation

---

### 11. `endsWith(String suffix)` - Check if Ends With

```java
String filename = "document.pdf";
String url = "https://example.com";

boolean ends1 = filename.endsWith(".pdf");      // true
boolean ends2 = filename.endsWith(".doc");      // false
boolean ends3 = url.endsWith(".com");           // true

/**
 * Real-world use case:
 * - File type checking
 * - Domain validation
 * - String suffix checking
 */

// Example: Image file validation
public boolean isImageFile(String filename) {
    String[] imageExtensions = {".jpg", ".jpeg", ".png", ".gif", ".bmp"};
    for (String ext : imageExtensions) {
        if (filename.toLowerCase().endsWith(ext)) {
            return true;
        }
    }
    return false;
}

// Better: Using stream
public boolean isImageFileBetter(String filename) {
    return Stream.of(".jpg", ".jpeg", ".png", ".gif", ".bmp")
        .anyMatch(ext -> filename.toLowerCase().endsWith(ext));
}

// Example: Commercial email domain
public boolean isCommercialEmail(String email) {
    return email.endsWith("@gmail.com") ||
           email.endsWith("@yahoo.com") ||
           email.endsWith("@outlook.com");
}

// Example: Database backup file
public boolean isBackupFile(String filename) {
    return filename.endsWith(".bak") ||
           filename.endsWith(".backup") ||
           filename.endsWith(".backup.zip");
}
```

**When to use:**
- File extension checking
- Domain validation
- Suffix validation
- File type filtering

---

## String Manipulation Methods

### 12. `substring(int beginIndex)` - Extract Substring from Index

```java
String text = "Hello World";

String sub1 = text.substring(0);        // "Hello World"
String sub2 = text.substring(6);        // "World"
String sub3 = text.substring(3);        // "lo World"

/**
 * Real-world use case:
 * - String extraction
 * - Removing prefix
 * - Text parsing
 */

// Example: Remove "www." from domain
public String removeDomainPrefix(String domain) {
    if (domain.startsWith("www.")) {
        return domain.substring(4);
    }
    return domain;
}
// removeDomainPrefix("www.example.com") → "example.com"

// Example: Extract account number (skip prefix)
public String getAccountNumber(String accountId) {
    // Assuming format: "ACC-123456"
    if (accountId.contains("-")) {
        int dashIndex = accountId.indexOf("-");
        return accountId.substring(dashIndex + 1);
    }
    return accountId;
}

// Example: Skip HTTP protocol
public String getUrlWithoutProtocol(String url) {
    if (url.startsWith("https://")) {
        return url.substring(8);
    } else if (url.startsWith("http://")) {
        return url.substring(7);
    }
    return url;
}
```

**When to use:**
- Removing prefixes
- String extraction
- Text parsing

---

### 13. `substring(int beginIndex, int endIndex)` - Extract Range

```java
String text = "Hello World";

String sub1 = text.substring(0, 5);     // "Hello"
String sub2 = text.substring(6, 11);    // "World"
String sub3 = text.substring(1, 4);     // "ell"

// Note: endIndex is exclusive

/**
 * Real-world use case:
 * - Extracting specific ranges
 * - Truncating strings
 * - Text processing
 */

// Example: Extract first N characters
public String getFirstNChars(String text, int n) {
    if (n > text.length()) {
        return text;
    }
    return text.substring(0, n);
}
// getFirstNChars("Hello World", 5) → "Hello"

// Example: Truncate with ellipsis
public String truncateText(String text, int maxLength) {
    if (text.length() <= maxLength) {
        return text;
    }
    return text.substring(0, maxLength - 3) + "...";
}
// truncateText("Hello World", 8) → "Hello..."

// Example: Extract middle portion
public String getMiddle(String text) {
    int length = text.length();
    int start = length / 4;
    int end = 3 * length / 4;
    return text.substring(start, end);
}

// Example: Parse date string (MM/DD/YYYY)
public void parseDate(String dateStr) {
    String month = dateStr.substring(0, 2);      // MM
    String day = dateStr.substring(3, 5);        // DD
    String year = dateStr.substring(6, 10);      // YYYY
    System.out.println("Month: " + month + ", Day: " + day + ", Year: " + year);
}
// parseDate("12/25/2026") → "Month: 12, Day: 25, Year: 2026"

// Example: Extract authorization token
public String extractToken(String authHeader) {
    // Format: "Bearer <token>"
    if (authHeader.startsWith("Bearer ")) {
        return authHeader.substring(7);
    }
    return "";
}
```

**When to use:**
- Range extraction
- Text truncation
- Parsing formatted data
- Substring filtering

---

### 14. `replace(char oldChar, char newChar)` - Replace Characters

```java
String text = "Hello World";

String replaced1 = text.replace('o', '0');      // "Hell0 W0rld"
String replaced2 = text.replace(' ', '_');      // "Hello_World"
String replaced3 = text.replace('x', 'y');      // "Hello World" (no 'x')

/**
 * Real-world use case:
 * - Character replacement
 * - Encoding/decoding
 * - Data transformation
 */

// Example: Phone number formatting
public String formatPhoneNumber(String phone) {
    return phone.replace(" ", "").replace("-", "");
}
// formatPhoneNumber("123-456-7890") → "1234567890"

// Example: CSV escaping
public String escapeCSV(String value) {
    return value.replace("\"", "\"\"");
}

// Example: URL safe string
public String makeUrlSafe(String text) {
    return text.replace(" ", "%20")
               .replace("&", "%26")
               .replace("=", "%3D");
}

// Example: Normalize whitespace
public String normalizeSpaces(String text) {
    return text.replace('\t', ' ')
               .replace('\n', ' ')
               .replace('\r', ' ');
}
```

**When to use:**
- Character replacement
- Data normalization
- URL encoding
- String escaping

---

### 15. `replace(CharSequence target, CharSequence replacement)` - Replace Substring

```java
String text = "Hello World, Hello Universe";

String replaced1 = text.replace("Hello", "Hi");     // "Hi World, Hi Universe"
String replaced2 = text.replace("World", "Java");   // "Hello Java, Hello Universe"
String replaced3 = text.replace("xyz", "abc");      // No change

/**
 * Real-world use case:
 * - String replacement
 * - Template processing
 * - Find and replace
 */

// Example: Simple template processing
public String processTemplate(String template, String name) {
    return template.replace("{{NAME}}", name);
}
// processTemplate("Hello {{NAME}}", "John") → "Hello John"

// Example: HTML entity encoding
public String encodeHtml(String text) {
    return text.replace("&", "&amp;")
               .replace("<", "&lt;")
               .replace(">", "&gt;")
               .replace("\"", "&quot;")
               .replace("'", "&#39;");
}

// Example: File path normalization
public String normalizePath(String path) {
    return path.replace("\\", "/");
}

// Example: Remove all occurrences
public String removeAllSpaces(String text) {
    return text.replace(" ", "");
}

// WARNING: replace() replaces ALL occurrences
// For first occurrence only, use replaceFirst()
```

**When to use:**
- Template processing
- HTML encoding
- Global replacements
- String transformation

---

### 16. `replaceAll(String regex, String replacement)` - Replace with Regex

```java
String text = "Hello123World456";

String replaced1 = text.replaceAll("[0-9]", "");      // "HelloWorld"
String replaced2 = text.replaceAll("[A-Z]", "");      // "ello123orld"
String replaced3 = text.replaceAll("\\d+", "X");      // "HelloXWorldX"

/**
 * Real-world use case:
 * - Regex-based replacement
 * - Data cleaning
 * - Pattern matching
 */

// Example: Remove all non-alphanumeric characters
public String removeSpecialChars(String text) {
    return text.replaceAll("[^a-zA-Z0-9]", "");
}
// removeSpecialChars("Hello@123#World!") → "Hello123World"

// Example: Extract numbers only
public String extractNumbers(String text) {
    return text.replaceAll("[^0-9]", "");
}
// extractNumbers("Phone: 123-456-7890") → "1234567890"

// Example: Normalize multiple spaces
public String normalizeSpaces(String text) {
    return text.replaceAll("\\s+", " ");
}
// normalizeSpaces("Hello    World") → "Hello World"

// Example: Remove HTML tags
public String stripHtml(String html) {
    return html.replaceAll("<[^>]*>", "");
}
// stripHtml("Hello <b>World</b>") → "Hello World"

// Example: Camel case to snake case
public String camelToSnakeCase(String text) {
    return text.replaceAll("([a-z])([A-Z])", "$1_$2").toLowerCase();
}
// camelToSnakeCase("firstName") → "first_name"

// Example: Phone number formatting
public String formatPhone(String phone) {
    String digits = phone.replaceAll("[^0-9]", "");
    return digits.replaceAll("(\\d{3})(\\d{3})(\\d{4})", "($1) $2-$3");
}
// formatPhone("1234567890") → "(123) 456-7890"

// WARNING: Regex can be expensive, use carefully in loops
```

**When to use:**
- Pattern-based replacement
- Data cleaning
- Regex matching
- Text transformation

---

### 17. `replaceFirst(String regex, String replacement)` - Replace First Match

```java
String text = "Hello World World";

String replaced1 = text.replaceFirst("World", "Java");    // "Hello Java World"
String replaced2 = text.replaceFirst("\\d", "X");         // "Hello World World"

/**
 * Real-world use case:
 * - Replacing first occurrence only
 * - Incremental replacement
 * - Pattern matching
 */

// Example: Replace first error in log
public String fixFirstError(String logContent, String errorPattern) {
    return logContent.replaceFirst(errorPattern, "[FIXED]");
}

// Example: Add prefix to first word
public String capitalizeFirst(String text) {
    return text.replaceFirst("^[a-z]", m -> m.group().toUpperCase());
}
// capitalizeFirst("hello world") → "Hello world"

// Example: Replace first space with comma
public String firstToComma(String text) {
    return text.replaceFirst(" ", ",");
}
// firstToComma("Hello World Java") → "Hello,World Java"
```

**When to use:**
- Single replacement
- First occurrence only
- Pattern-based replacement

---

## String Transformation Methods

### 18. `toUpperCase()` & `toLowerCase()` - Case Conversion

```java
String text = "Hello World";

String upper = text.toUpperCase();              // "HELLO WORLD"
String lower = text.toLowerCase();              // "hello world"

// With specific locale
String upperTR = text.toUpperCase(Locale.forLanguageTag("tr")); // Turkish

/**
 * Real-world use case:
 * - Case normalization
 * - Comparison
 * - Display formatting
 */

// Example: Case-insensitive comparison
public boolean isCommand(String input, String command) {
    return input.toLowerCase().equals(command.toLowerCase());
}
// isCommand("HELLO", "hello") → true

// Example: API key normalization
public String normalizeApiKey(String key) {
    return key.toUpperCase().replace(" ", "");
}

// Example: File name case conversion
public String convertToLowercase(String filename) {
    return filename.toLowerCase();
}

// Example: Menu item display
public String formatMenuItemName(String itemName) {
    return itemName.substring(0, 1).toUpperCase() + 
           itemName.substring(1).toLowerCase();
}
// formatMenuItemName("HELLO") → "Hello"
```

**When to use:**
- Case normalization
- Case-insensitive comparison
- Display formatting
- API standardization

---

### 19. `trim()` - Remove Leading/Trailing Whitespace

```java
String text1 = "  Hello World  ";
String text2 = "\tHello\n";

String trimmed1 = text1.trim();                 // "Hello World"
String trimmed2 = text2.trim();                 // "Hello"

/**
 * Real-world use case:
 * - User input cleaning
 * - Whitespace removal
 * - Data validation
 */

// Example: Validate non-empty input
public boolean isValidInput(String input) {
    return input != null && !input.trim().isEmpty();
}

// Example: Clean user data
public String cleanUserInput(String input) {
    return input.trim();
}

// Example: Parse CSV field
public String[] parseCSV(String line) {
    String[] fields = line.split(",");
    for (int i = 0; i < fields.length; i++) {
        fields[i] = fields[i].trim();
    }
    return fields;
}

// Example: Remove extra whitespace
public String normalizeText(String text) {
    return text.trim()
               .replaceAll("\\s+", " ");
}
// normalizeText("  Hello   World  ") → "Hello World"

// Java 11+ strip() methods
String stripped = text1.strip();                // Same as trim()
String leadingStripped = text1.stripLeading(); // Remove leading whitespace
String trailingStripped = text1.stripTrailing();// Remove trailing whitespace
```

**When to use:**
- User input validation
- Data cleaning
- CSV parsing
- Whitespace removal

---

## String Formatting Methods

### 20. `format(String format, Object... args)` - Format String

```java
String name = "John";
int age = 30;
double salary = 50000.50;

String formatted1 = String.format("Name: %s, Age: %d", name, age);
// "Name: John, Age: 30"

String formatted2 = String.format("Salary: $%.2f", salary);
// "Salary: $50000.50"

String formatted3 = String.format("%d + %d = %d", 2, 3, 5);
// "2 + 3 = 5"

/**
 * Real-world use case:
 * - String formatting
 * - Logging
 * - Report generation
 * - Output formatting
 */

// Format Specifiers:
// %s - String
// %d - Integer
// %f - Float/Double
// %x - Hexadecimal
// %b - Boolean
// %n - Newline

// Example: Generate user report
public String generateUserReport(String name, int userId, double balance) {
    return String.format(
        "User Report%n" +
        "Name: %s%n" +
        "ID: %d%n" +
        "Balance: $%.2f",
        name, userId, balance
    );
}

// Example: Log message formatting
public void logError(String component, int errorCode, String message) {
    String log = String.format("[ERROR] %s - Code: %d - Message: %s", 
                               component, errorCode, message);
    System.err.println(log);
}

// Example: Right-align numbers
String formatted = String.format("%10d", 42);     // "        42"

// Example: Left-align with padding
String formatted2 = String.format("%-10s", "Hello"); // "Hello     "

// Example: Zero-padded numbers
String formatted3 = String.format("%05d", 42);    // "00042"

// Example: Hexadecimal conversion
String hex = String.format("0x%X", 255);         // "0xFF"

// Example: Percentage formatting
String percentage = String.format("%.1f%%", 95.5); // "95.5%"
```

**When to use:**
- String formatting
- Number formatting
- Logging
- Report generation

---

### 21. `valueOf(Object obj)` - Convert to String

```java
int number = 42;
double decimal = 3.14;
boolean flag = true;
char letter = 'A';

String str1 = String.valueOf(number);           // "42"
String str2 = String.valueOf(decimal);          // "3.14"
String str3 = String.valueOf(flag);             // "true"
String str4 = String.valueOf(letter);           // "A"

/**
 * Real-world use case:
 * - Type conversion
 * - Object to string conversion
 * - Data serialization
 */

// Example: Convert array to string
int[] numbers = {1, 2, 3, 4, 5};
String arrayStr = String.valueOf(Arrays.toString(numbers));
// "[1, 2, 3, 4, 5]"

// Example: Null-safe conversion
public String safeToString(Object obj) {
    return String.valueOf(obj); // Returns "null" if obj is null
}

// Example: Number validation
public boolean isNumeric(String text) {
    try {
        Integer.parseInt(text);
        return true;
    } catch (NumberFormatException e) {
        return false;
    }
}

// Example: Configuration value conversion
public String getConfigValue(Object value) {
    return value == null ? "default" : String.valueOf(value);
}
```

**When to use:**
- Type conversion
- Object to string conversion
- Null-safe conversion
- Serialization

---

## Advanced String Operations

### 22. `split(String regex)` - Split String by Delimiter

```java
String csv = "apple,banana,orange,grape";
String sentence = "Hello World Java Programming";
String path = "/home/user/documents/file.txt";

String[] fruits = csv.split(",");
// ["apple", "banana", "orange", "grape"]

String[] words = sentence.split(" ");
// ["Hello", "World", "Java", "Programming"]

String[] pathParts = path.split("/");
// ["", "home", "user", "documents", "file.txt"]

// Split with limit
String[] limited = csv.split(",", 2);
// ["apple", "banana,orange,grape"]

/**
 * Real-world use case:
 * - CSV parsing
 * - Sentence tokenization
 * - Path parsing
 * - Data extraction
 */

// Example: Parse CSV line
public String[] parseCSVLine(String line) {
    return line.split(",(?=([^\"]*\"[^\"]*\")*[^\"]*$)");
    // Handles quoted fields with commas
}

// Example: Extract email parts
public void extractEmailParts(String email) {
    String[] parts = email.split("@");
    String localPart = parts[0];
    String domain = parts[1];
    System.out.println("User: " + localPart + ", Domain: " + domain);
}

// Example: Parse IP address
public void parseIP(String ip) {
    String[] octets = ip.split("\\.");
    for (int i = 0; i < octets.length; i++) {
        System.out.println("Octet " + (i + 1) + ": " + octets[i]);
    }
}

// Example: Parse query parameters
public Map<String, String> parseQueryParams(String queryString) {
    Map<String, String> params = new HashMap<>();
    String[] pairs = queryString.split("&");
    for (String pair : pairs) {
        String[] keyValue = pair.split("=");
        params.put(keyValue[0], keyValue.length > 1 ? keyValue[1] : "");
    }
    return params;
}
// parseQueryParams("name=John&age=30&city=NYC") → {name=John, age=30, city=NYC}

// Example: Split by multiple delimiters
public String[] splitByMultiple(String text) {
    return text.split("[,;|]"); // Split by comma, semicolon, or pipe
}

// WARNING: Metacharacters in regex (., *, +, ?, ^, $, |, \, etc.) need escaping
// Wrong: split(".");
// Correct: split("\\.");
```

**When to use:**
- CSV parsing
- Sentence tokenization
- Path parsing
- Configuration parsing

---

### 23. `join(CharSequence delimiter, CharSequence... elements)` - Join Strings

```java
String[] fruits = {"apple", "banana", "orange"};
List<String> colors = Arrays.asList("red", "green", "blue");

String result1 = String.join(",", fruits);
// "apple,banana,orange"

String result2 = String.join(" - ", colors);
// "red - green - blue"

String result3 = String.join("\n", "Line 1", "Line 2", "Line 3");
// "Line 1\nLine 2\nLine 3"

/**
 * Real-world use case:
 * - Creating delimited strings
 * - CSV generation
 * - URL building
 * - Log formatting
 */

// Example: Create CSV line
public String toCSV(String... fields) {
    return String.join(",", fields);
}
// toCSV("John", "30", "USA") → "John,30,USA"

// Example: Create URL path
public String buildPath(String... segments) {
    return String.join("/", segments);
}
// buildPath("api", "v1", "users", "123") → "api/v1/users/123"

// Example: Create log entry
public String formatLog(String timestamp, String level, String message) {
    return String.join(" | ", timestamp, level, message);
}
// formatLog("2026-06-09", "ERROR", "File not found") → "2026-06-09 | ERROR | File not found"

// Example: Build SQL query (with List)
public String buildInsert(String table, List<String> columns, List<String> values) {
    String columnList = String.join(",", columns);
    String valueList = String.join(",", values);
    return String.format("INSERT INTO %s (%s) VALUES (%s)", table, columnList, valueList);
}

// Example: Create tag string
public String createTagString(List<String> tags) {
    return String.join(", ", tags);
}
// createTagString(["java", "coding", "tutorial"]) → "java, coding, tutorial"
```

**When to use:**
- Creating delimited strings
- CSV generation
- URL path building
- Log formatting
- Array to string conversion

---

### 24. `matches(String regex)` - Check if Matches Pattern

```java
String email1 = "john@example.com";
String email2 = "invalid.email";

String emailRegex = "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$";

boolean isEmail1 = email1.matches(emailRegex);  // true
boolean isEmail2 = email2.matches(emailRegex);  // false

/**
 * Real-world use case:
 * - Email validation
 * - Phone number validation
 * - Password strength checking
 * - Format validation
 */

// Example: Email validation
public boolean isValidEmail(String email) {
    String emailRegex = "^[\\w.-]+@[\\w.-]+\\.\\w+$";
    return email.matches(emailRegex);
}

// Example: Phone number validation
public boolean isValidPhone(String phone) {
    return phone.matches("\\d{3}-\\d{3}-\\d{4}");
}
// Valid: "123-456-7890"

// Example: Zip code validation
public boolean isValidZipCode(String zip) {
    return zip.matches("\\d{5}(-\\d{4})?");
}
// Valid: "12345" or "12345-6789"

// Example: Strong password validation
public boolean isStrongPassword(String password) {
    String regex = "^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]{8,}$";
    return password.matches(regex);
}
// Requirements: min 8 chars, 1 uppercase, 1 lowercase, 1 digit, 1 special char

// Example: Hexadecimal validation
public boolean isHexColor(String color) {
    return color.matches("#[0-9A-Fa-f]{6}");
}
// Valid: "#FF5733"

// WARNING: matches() compiles regex each time, use Pattern for repeated checks
```

**When to use:**
- Email validation
- Phone number validation
- Pattern matching
- Format validation

---

## String Immutability & Performance

### Important Concepts

```java
/**
 * String Immutability
 * 
 * Strings are IMMUTABLE in Java
 * Every operation creates a new String object
 */

// Example 1: String immutability
String original = "Hello";
String modified = original.concat(" World");
System.out.println(original); // Still "Hello"
System.out.println(modified); // "Hello World"

// Example 2: String pool
String str1 = "Hello";
String str2 = "Hello";
System.out.println(str1 == str2); // true (same reference in pool)

String str3 = new String("Hello");
System.out.println(str1 == str3); // false (different objects)
System.out.println(str1.equals(str3)); // true (same content)

// Example 3: Performance consideration - String concatenation in loop
// ❌ INEFFICIENT
String result = "";
for (int i = 0; i < 1000; i++) {
    result += "item" + i + ", ";  // Creates 1000 new String objects!
}

// ✅ EFFICIENT - Use StringBuilder
StringBuilder builder = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    builder.append("item").append(i).append(", ");
}
String result = builder.toString(); // Single String object

// Example 4: StringBuffer (thread-safe StringBuilder)
StringBuffer buffer = new StringBuffer();
buffer.append("Hello");
buffer.append(" ");
buffer.append("World");
String result = buffer.toString(); // "Hello World"

/**
 * StringBuffer vs StringBuilder
 * 
 * StringBuffer:
 * - Thread-safe (synchronized)
 * - Slower
 * - Use in multi-threaded environment
 * 
 * StringBuilder:
 * - NOT thread-safe
 * - Faster
 * - Use in single-threaded environment
 */

// Example: StringBuilder for building complex strings
public String generateReport(List<User> users) {
    StringBuilder report = new StringBuilder();
    report.append("========== USER REPORT ==========\n");
    
    for (User user : users) {
        report.append("Name: ").append(user.getName()).append("\n");
        report.append("Email: ").append(user.getEmail()).append("\n");
        report.append("Age: ").append(user.getAge()).append("\n");
        report.append("-----------------------------------\n");
    }
    
    report.append("Total Users: ").append(users.size()).append("\n");
    return report.toString();
}
```

---

## Real-World Examples

### Example 1: Email Validation and Extraction

```java
public class EmailValidator {
    
    public boolean isValidEmail(String email) {
        if (email == null || email.trim().isEmpty()) {
            return false;
        }
        
        String emailRegex = "^[\\w.-]+@[\\w.-]+\\.\\w+$";
        return email.matches(emailRegex);
    }
    
    public String extractUsername(String email) {
        int atIndex = email.indexOf("@");
        if (atIndex == -1) {
            return "";
        }
        return email.substring(0, atIndex);
    }
    
    public String extractDomain(String email) {
        int atIndex = email.indexOf("@");
        if (atIndex == -1) {
            return "";
        }
        return email.substring(atIndex + 1);
    }
    
    public boolean isCompanyEmail(String email, String companyDomain) {
        return isValidEmail(email) && 
               extractDomain(email).equalsIgnoreCase(companyDomain);
    }
}

// Usage
EmailValidator validator = new EmailValidator();
validator.isValidEmail("john@example.com");        // true
validator.extractUsername("john@example.com");     // "john"
validator.extractDomain("john@example.com");       // "example.com"
```

---

### Example 2: Log Level Parsing

```java
public class LogProcessor {
    
    public enum LogLevel {
        INFO, WARN, ERROR, DEBUG
    }
    
    public LogLevel extractLogLevel(String logLine) {
        if (logLine.contains("[ERROR]")) {
            return LogLevel.ERROR;
        } else if (logLine.contains("[WARN]")) {
            return LogLevel.WARN;
        } else if (logLine.contains("[DEBUG]")) {
            return LogLevel.DEBUG;
        }
        return LogLevel.INFO;
    }
    
    public String extractMessage(String logLine) {
        String[] markers = {"[ERROR]", "[WARN]", "[DEBUG]", "[INFO]"};
        for (String marker : markers) {
            if (logLine.contains(marker)) {
                int endIndex = logLine.indexOf(marker) + marker.length();
                return logLine.substring(endIndex).trim();
            }
        }
        return logLine;
    }
    
    public void processLog(String logLine) {
        LogLevel level = extractLogLevel(logLine);
        String message = extractMessage(logLine);
        
        switch(level) {
            case ERROR:
                System.err.println("ERROR: " + message);
                break;
            case WARN:
                System.out.println("WARNING: " + message);
                break;
            default:
                System.out.println("INFO: " + message);
        }
    }
}

// Usage
LogProcessor processor = new LogProcessor();
processor.processLog("[ERROR] Database connection failed");
processor.processLog("[WARN] Deprecated method used");
```

---

### Example 3: CSV Parser

```java
public class CSVParser {
    
    public List<String> parseCSVLine(String line) {
        List<String> fields = new ArrayList<>();
        StringBuilder field = new StringBuilder();
        boolean inQuotes = false;
        
        for (int i = 0; i < line.length(); i++) {
            char c = line.charAt(i);
            
            if (c == '"') {
                inQuotes = !inQuotes;
            } else if (c == ',' && !inQuotes) {
                fields.add(field.toString().trim());
                field = new StringBuilder();
            } else {
                field.append(c);
            }
        }
        
        fields.add(field.toString().trim());
        return fields;
    }
    
    public List<Map<String, String>> parseCSV(List<String> lines) {
        List<Map<String, String>> records = new ArrayList<>();
        
        if (lines.isEmpty()) {
            return records;
        }
        
        List<String> headers = parseCSVLine(lines.get(0));
        
        for (int i = 1; i < lines.size(); i++) {
            List<String> values = parseCSVLine(lines.get(i));
            Map<String, String> record = new HashMap<>();
            
            for (int j = 0; j < headers.size(); j++) {
                if (j < values.size()) {
                    record.put(headers.get(j), values.get(j));
                }
            }
            
            records.add(record);
        }
        
        return records;
    }
}

// Usage
CSVParser parser = new CSVParser();
List<String> lines = Arrays.asList(
    "Name,Email,Age",
    "John,john@example.com,30",
    "Jane,jane@example.com,25"
);
List<Map<String, String>> data = parser.parseCSV(lines);
// Results in list of maps with Name, Email, Age as keys
```

---

### Example 4: URL Builder

```java
public class URLBuilder {
    
    private String protocol = "https";
    private String domain = "";
    private String path = "";
    private Map<String, String> queryParams = new LinkedHashMap<>();
    
    public URLBuilder protocol(String protocol) {
        this.protocol = protocol;
        return this;
    }
    
    public URLBuilder domain(String domain) {
        this.domain = domain;
        return this;
    }
    
    public URLBuilder path(String... segments) {
        this.path = "/" + String.join("/", segments);
        return this;
    }
    
    public URLBuilder param(String key, String value) {
        queryParams.put(key, value);
        return this;
    }
    
    public String build() {
        StringBuilder url = new StringBuilder();
        url.append(protocol).append("://").append(domain);
        
        if (!path.isEmpty()) {
            url.append(path);
        }
        
        if (!queryParams.isEmpty()) {
            url.append("?");
            List<String> params = new ArrayList<>();
            for (Map.Entry<String, String> entry : queryParams.entrySet()) {
                params.add(entry.getKey() + "=" + entry.getValue());
            }
            url.append(String.join("&", params));
        }
        
        return url.toString();
    }
}

// Usage
String url = new URLBuilder()
    .protocol("https")
    .domain("api.example.com")
    .path("users", "profile")
    .param("userId", "123")
    .param("format", "json")
    .build();
// Result: "https://api.example.com/users/profile?userId=123&format=json"
```

---

### Example 5: Password Validator

```java
public class PasswordValidator {
    
    public enum PasswordStrength {
        WEAK("At least 6 characters"),
        MEDIUM("At least 8 characters with uppercase and lowercase"),
        STRONG("At least 10 characters with uppercase, lowercase, digits, and special chars");
        
        private String requirement;
        
        PasswordStrength(String requirement) {
            this.requirement = requirement;
        }
    }
    
    public PasswordStrength checkStrength(String password) {
        if (password.length() < 6) {
            return PasswordStrength.WEAK;
        }
        
        if (password.length() < 8 ||
            !password.matches(".*[A-Z].*") ||
            !password.matches(".*[a-z].*")) {
            return PasswordStrength.WEAK;
        }
        
        if (password.length() < 10 ||
            !password.matches(".*\\d.*") ||
            !password.matches(".*[@$!%*?&].*")) {
            return PasswordStrength.MEDIUM;
        }
        
        return PasswordStrength.STRONG;
    }
    
    public boolean isValidPassword(String password) {
        return checkStrength(password) != PasswordStrength.WEAK;
    }
    
    public List<String> getValidationErrors(String password) {
        List<String> errors = new ArrayList<>();
        
        if (password == null || password.isEmpty()) {
            errors.add("Password cannot be empty");
            return errors;
        }
        
        if (password.length() < 8) {
            errors.add("Password must be at least 8 characters long");
        }
        
        if (!password.matches(".*[A-Z].*")) {
            errors.add("Password must contain at least one uppercase letter");
        }
        
        if (!password.matches(".*[a-z].*")) {
            errors.add("Password must contain at least one lowercase letter");
        }
        
        if (!password.matches(".*\\d.*")) {
            errors.add("Password must contain at least one digit");
        }
        
        if (!password.matches(".*[@$!%*?&].*")) {
            errors.add("Password must contain at least one special character (@$!%*?&)");
        }
        
        return errors;
    }
}

// Usage
PasswordValidator validator = new PasswordValidator();
validator.checkStrength("pass123");        // WEAK
validator.checkStrength("Pass123");        // MEDIUM
validator.checkStrength("Pass@123");       // STRONG
validator.getValidationErrors("weak");     // List of errors
```

---

## Summary Table: When to Use Each Method

| Method | Use Case | Example |
|--------|----------|---------|
| `length()` | Get string size | Validate input length |
| `charAt()` | Access single character | Check first letter |
| `isEmpty()` | Check if empty | Form validation |
| `equals()` | Exact match | Password verification |
| `equalsIgnoreCase()` | Case-insensitive match | Command processing |
| `compareTo()` | Ordering | Sorting arrays |
| `indexOf()` | Find position | Email parsing |
| `lastIndexOf()` | Find last position | File extension |
| `contains()` | Check substring | Content filtering |
| `startsWith()` | Check prefix | Protocol validation |
| `endsWith()` | Check suffix | File type checking |
| `substring()` | Extract portion | Remove prefix |
| `replace()` | Replace characters | Data normalization |
| `replaceAll()` | Regex replacement | Data cleaning |
| `toUpperCase()` | Convert to uppercase | Normalization |
| `toLowerCase()` | Convert to lowercase | Case-insensitive |
| `trim()` | Remove whitespace | Input cleaning |
| `format()` | Format string | Report generation |
| `split()` | Split by delimiter | CSV parsing |
| `join()` | Join with delimiter | CSV creation |
| `matches()` | Regex validation | Email validation |

---

## Conclusion

Java String methods provide powerful tools for text processing. Key takeaways:

1. **Strings are immutable** — Every operation creates a new object
2. **Use StringBuilder** — For concatenation in loops
3. **Choose appropriate methods** — Use specialized methods for specific tasks
4. **Regex when needed** — Use `matches()` and `replaceAll()` for complex patterns
5. **Performance matters** — Avoid unnecessary string operations in loops

